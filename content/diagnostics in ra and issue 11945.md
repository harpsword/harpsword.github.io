+++

title = "Diagnositcs Calculation in Rust Analyzer and Issue 11945 Fix Solution"
date = 2022-05-18

[taxonomies]
categories = ["Rust"]
tags = ["Rust", "Rust Analyzer"]
+++

# 背景

在使用Rust Analyzer(下文中称RA)的过程中，经常会出现某些代码飘红的情况，表示这些代码存在问题。RA通过LSP协议中的publishDiagnositcs接口把代码诊断的结果告诉了VS Code。

RA中的诊断信息有两种来源，第一种是调用rustc来分析代码得到，另一种是RA内部代码分析所得。

## 调用rustc分析代码
RA中调用rustc分析代码的过程可以分为两部分，第一部分是调用相同的库来分析代码，并把结果通过Event的形式发送出去；第二部分在main_loop中，接收到Event之后，发现是Event::Flycheck，则对其分析结果进行转换，转换为lsp::Diagnostic格式后，发送给VS Code。

```rust
enum Event {
    Lsp(lsp_server::Message),
    Task(Task),
    Vfs(vfs::loader::Message),
    Flycheck(flycheck::Message),
}
```

### 调用 rustc

在RA中这一部分代码取名为flycheck，这个是借用了之前emacs中的开源包名。详细的调用flycheck的代码在 crates/flycheck/src/lib.rs, FlycheckActor.run方法。这里的实现常用Actor模式，下面的CargoActor、CargoHandle就是一个例子。

```rust
struct CargoActor {
    sender: Sender<CargoMessage>,
}
impl CargoActor {
    fn new(sender: Sender<CargoMessage>) -> CargoActor {
        CargoActor { sender }
    }

    fn run(self, command: Command) -> io::Result<()> {
        // 实际上的运行代码，运行完成会把数据通过 sender发送出去
    }
}

struct CargoHandle {
    thread: jod_thread::JoinHandle<io::Result<()>>,
    receiver: Receiver<CargoMessage>,
}

impl CargoHandle {
    // 负责初始化actor，并提交到作业线程池中，
    fn spawn(command: Command) -> CargoHandle {
        let (sender, receiver) = unbounded();
        let actor = CargoActor::new(sender);
        let thread = jod_thread::Builder::new()
            .name("CargoHandle".to_owned())
            .spawn(move || actor.run(command))
            .expect("failed to spawn thread");
        CargoHandle { thread, receiver }
    }

    fn join(self) -> io::Result<()> {
        self.thread.join()
    }
}
```

也有下面这种，FlycheckHandle、FlycheckActor，FlycheckActor中同时有sender和receiver，receiver负责接收来自Handle的指令，sender负责把结果传到外面。

```rust
#[derive(Debug)]
pub struct FlycheckHandle {
    // XXX: drop order is significant
    sender: Sender<Restart>,
    _thread: jod_thread::JoinHandle,
}

impl FlycheckHandle {
    pub fn spawn(
        id: usize,
        sender: Box<dyn Fn(Message) + Send>,
        config: FlycheckConfig,
        workspace_root: AbsPathBuf,
    ) -> FlycheckHandle {
        // 在这里把 sender给传下去了
        let actor = FlycheckActor::new(id, sender, config, workspace_root);
        let (sender, receiver) = unbounded::<Restart>();
        // 初始化actor，到那时run中传的参数是 receiver
        let thread = jod_thread::Builder::new()
            .name("Flycheck".to_owned())
            .spawn(move || actor.run(receiver))
            .expect("failed to spawn thread");
        FlycheckHandle { sender, _thread: thread }
    }

    /// Schedule a re-start of the cargo check worker.
    pub fn update(&self) {
        self.sender.send(Restart).unwrap();
    }
}

struct FlycheckActor {
    id: usize,
    sender: Box<dyn Fn(Message) + Send>,
    config: FlycheckConfig,
    workspace_root: AbsPathBuf,
    cargo_handle: Option<CargoHandle>,
}

enum Event {
    Restart(Restart),
    CheckEvent(Option<CargoMessage>),
}

impl FlycheckActor {
    fn new(
        id: usize,
        sender: Box<dyn Fn(Message) + Send>,
        config: FlycheckConfig,
        workspace_root: AbsPathBuf,
    ) -> FlycheckActor {
        FlycheckActor { id, sender, config, workspace_root, cargo_handle: None }
    }
    fn progress(&self, progress: Progress) {
        self.send(Message::Progress { id: self.id, progress });
    }
    fn next_event(&self, inbox: &Receiver<Restart>) -> Option<Event> {
        let check_chan = self.cargo_handle.as_ref().map(|cargo| &cargo.receiver);
        select! {
            recv(inbox) -> msg => msg.ok().map(Event::Restart),
            recv(check_chan.unwrap_or(&never())) -> msg => Some(Event::CheckEvent(msg.ok())),
        }
    }
    fn run(mut self, inbox: Receiver<Restart>) {
    }
}
```



### 转换
RA的main_loop也是一个处理消息的，Flycheck的结果也是通过这种方式传递出来，在main_loop中负责解析并处理对应的EVENT。对应的EVENT处理代码如下所示，在这里只截取了diagnostics的部分，会使用crate::diagnostics::to_proto::map_rust_diagnostic_to_lsp方法来转换diagostics，转换之后保存在self.diagnostics中。
```rust
Event::Flycheck(mut task) => {
    let _p = profile::span("GlobalState::handle_event/flycheck");
    loop {
        match task {
            flycheck::Message::AddDiagnostic { workspace_root, diagnostic } => {
                let snap = self.snapshot();
                let diagnostics =
                    crate::diagnostics::to_proto::map_rust_diagnostic_to_lsp(
                        &self.config.diagnostics_map(),
                        &diagnostic,
                        &workspace_root,
                        &snap,
                    );
                for diag in diagnostics {
                    match url_to_file_id(&self.vfs.read().0, &diag.url) {
                        Ok(file_id) => self.diagnostics.add_check_diagnostic(
                            file_id,
                            diag.diagnostic,
                            diag.fix,
                        ),
                        Err(err) => {
                            tracing::error!(
                                "File with cargo diagnostic not found in VFS: {}",
                                err
                            );
                        }
                    };
                }
            }
        }
    }
}
```

RA会在处理Event之后统一查看 self.diagnostics中是否有新数据需要发送到Client，最后这一步对不同diagnositc数据来源是相同的。
```rust
if let Some(diagnostic_changes) = self.diagnostics.take_changes() {
    for file_id in diagnostic_changes {
        let db = self.analysis_host.raw_database();
        let source_root = db.file_source_root(file_id);
        if db.source_root(source_root).is_library {
            // Only publish diagnostics for files in the workspace, not from crates.io deps
            // or the sysroot.
            // While theoretically these should never have errors, we have quite a few false
            // positives particularly in the stdlib, and those diagnostics would stay around
            // forever if we emitted them here.
            continue;
        }

        let url = file_id_to_url(&self.vfs.read().0, file_id);
        let diagnostics = self.diagnostics.diagnostics_for(file_id).cloned().collect();
        let version = from_proto::vfs_path(&url)
            .map(|path| self.mem_docs.get(&path).map(|it| it.version))
            .unwrap_or_default();

        self.send_notification::<lsp_types::notification::PublishDiagnostics>(
            lsp_types::PublishDiagnosticsParams { uri: url, diagnostics, version },
        );
    }
}
```

## 内部分析代码
RA内部的诊断代码同样分为两部分，一部分计算diagnostics，另一部分收到数据后保存到self.diagnostics。

### 计算diagnostics

对应的入口代码如下所示
```rust
fn update_diagnostics(&mut self) {
    let subscriptions = self
        .mem_docs
        .iter()
        .map(|path| self.vfs.read().0.file_id(path).unwrap())
        .collect::<Vec<_>>();

    tracing::trace!("updating notifications for {:?}", subscriptions);

    let snapshot = self.snapshot();
    self.task_pool.handle.spawn(move || {
        let diagnostics = subscriptions
            .into_iter()
            .filter_map(|file_id| {
                handlers::publish_diagnostics(&snapshot, file_id)
                    .map_err(|err| {
                        if !is_cancelled(&*err) {
                            tracing::error!("failed to compute diagnostics: {:?}", err);
                        }
                    })
                    .ok()
                    .map(|diags| (file_id, diags))
            })
            .collect::<Vec<_>>();
        Task::Diagnostics(diagnostics)
    })
}
```

实际上计算的代码在crates/ide-diagnostics/src/lib.rs, fn diagnostics。
```rust
pub fn diagnostics(
    db: &RootDatabase,
    config: &DiagnosticsConfig,
    resolve: &AssistResolveStrategy,
    file_id: FileId,
) -> Vec<Diagnostic> {
    let _p = profile::span("diagnostics");
    let sema = Semantics::new(db);
    // 解析遇到的错误会保存在 parse.errors中
    let parse = db.parse(file_id);
    let mut res = Vec::new();

    // [#34344] Only take first 128 errors to prevent slowing down editor/ide, the number 128 is chosen arbitrarily.
    // 把解析遇到的错误也提取出来
    res.extend(
        parse.errors().iter().take(128).map(|err| {
            Diagnostic::new("syntax-error", format!("Syntax Error: {}", err), err.range())
        }),
    );

    for node in parse.tree().syntax().descendants() {
        handlers::useless_braces::useless_braces(&mut res, file_id, &node);
        handlers::field_shorthand::field_shorthand(&mut res, file_id, &node);
    }

    let module = sema.to_module_def(file_id);

    let ctx = DiagnosticsContext { config, sema, resolve };
    if module.is_none() {
        handlers::unlinked_file::unlinked_file(&ctx, &mut res, file_id);
    }

    let mut diags = Vec::new();
    // 从各个module开始计算diagnostics
    // 在子节点那会根据节点类型调用对应节点的diagnostics方法
    if let Some(m) = module {
        m.diagnostics(db, &mut diags)
    }

    for diag in diags {
        #[rustfmt::skip]
        let d = match diag {
            AnyDiagnostic::BreakOutsideOfLoop(d) => handlers::break_outside_of_loop::break_outside_of_loop(&ctx, &d),
            // ...
            AnyDiagnostic::InvalidDeriveTarget(d) => handlers::invalid_derive_target::invalid_derive_target(&ctx, &d),

            AnyDiagnostic::InactiveCode(d) => match handlers::inactive_code::inactive_code(&ctx, &d) {
                Some(it) => it,
                None => continue,
            }
        };
        res.push(d)
    }

    res.retain(|d| {
        !ctx.config.disabled.contains(d.code.as_str())
            && !(ctx.config.disable_experimental && d.experimental)
    });

    res
}
```

语法验证的代码(crates/syntax/src/validation.rs)
```rust
pub(crate) fn validate(root: &SyntaxNode) -> Vec<SyntaxError> {
    // FIXME:
    // * Add unescape validation of raw string literals and raw byte string literals
    // * Add validation of doc comments are being attached to nodes

    let mut errors = Vec::new();
    for node in root.descendants() {
        match_ast! {
            match node {
                ast::Literal(it) => validate_literal(it, &mut errors),
                ast::Const(it) => validate_const(it, &mut errors),
                ast::BlockExpr(it) => block::validate_block_expr(it, &mut errors),
                ast::FieldExpr(it) => validate_numeric_name(it.name_ref(), &mut errors),
                ast::RecordExprField(it) => validate_numeric_name(it.name_ref(), &mut errors),
                ast::Visibility(it) => validate_visibility(it, &mut errors),
                ast::RangeExpr(it) => validate_range_expr(it, &mut errors),
                ast::PathSegment(it) => validate_path_keywords(it, &mut errors),
                ast::RefType(it) => validate_trait_object_ref_ty(it, &mut errors),
                ast::PtrType(it) => validate_trait_object_ptr_ty(it, &mut errors),
                ast::FnPtrType(it) => validate_trait_object_fn_ptr_ret_ty(it, &mut errors),
                ast::MacroRules(it) => validate_macro_rules(it, &mut errors),
                ast::LetExpr(it) => validate_let_expr(it, &mut errors),
                _ => (),
            }
        }
    }
    errors
}
```

### 数据收集

和flycheck类似，在main_loop中收到Task::Diagnostics消息后，把对应的disgnostics写入到self.diagnostics中。
```rust
Event::Task(mut task) => {
    let _p = profile::span("GlobalState::handle_event/task");
    let mut prime_caches_progress = Vec::new();
    loop {
        match task {
            Task::Response(response) => self.respond(response),
            Task::Diagnostics(diagnostics_per_file) => {
                for (file_id, diagnostics) in diagnostics_per_file {
                    self.diagnostics.set_native_diagnostics(file_id, diagnostics)
                }
            }
            // ...
        }
        // ..
    }
}
```

# Issue 详情

## 背景

出现问题的代码属于flycheck得到的diagnostics转换成lsp_server::diagnostics的过程。在这个过程中，需要将flycheck::Diagnostic中关于问题代码的范围转换为lsp_types::Diagnostic中的Range。VS Code可以采用UTF-8或者UTF-16编码，确定编码之后，编辑器上的字符虽然显示出来只有一个，但是可能会占据多个位置；在UTF-16编码下，一个`𐐀`会占用两个位置。而flycheck得到的Range都是按照字符来进行计数的，无法直接映射到VS Code的位置。

另外，RA内部的Offset计算是按照字节进行的。比如一个字符在UTF-8编码下需要两个字节来表示，那么在RA内部这个字符会占用两个位置。

## 详细例子

在以下示例代码中，存在用UTF-8格式下需要多个字节来表示的字符𐐀，在这种情况下，RA给出的范围是(17, 23)，而实际上应该是(17, 27)。对应的截图![图片](https://user-images.githubusercontent.com/308347/162568089-c6e77199-bc8c-43b7-934b-fd5b19b7c16b.png)
```rust
fn main() {
    let x: u32 = "𐐀𐐀𐐀𐐀";
}
```

## 修复方法

flycheck给的数据格式如下所示，可以利用column和原始的字符串来计算对应的lsp_type::range。
```rust
pub struct DiagnosticSpan {
    /// The file name or the macro name this diagnostic comes from.
    pub file_name: String,
    /// The byte offset in the file where this diagnostic starts from.
    pub byte_start: u32,
    /// The byte offset in the file where this diagnostic ends.
    pub byte_end: u32,
    /// 1-based. The line in the file.
    pub line_start: usize,
    /// 1-based. The line in the file.
    pub line_end: usize,
    /// 1-based, character offset.
    pub column_start: usize,
    /// 1-based, character offset.
    pub column_end: usize,
    /// ...
    /// Source text from the start of line_start to the end of line_end.
    pub text: Vec<DiagnosticSpanLine>,
}
```
当然需要考虑VS Code这边使用的编码，修改后的代码如下所示:
```rust
/// Converts a Rust span to a LSP location
fn location(
    config: &DiagnosticsMapConfig,
    workspace_root: &AbsPath,
    span: &DiagnosticSpan,
    snap: &GlobalStateSnapshot,
) -> lsp_types::Location {
    let file_name = resolve_path(config, workspace_root, &span.file_name);
    let uri = url_from_abs_path(&file_name);

    let range = {
        let offset_encoding = snap.config.offset_encoding();
        lsp_types::Range::new(
            position(&offset_encoding, span, span.line_start, span.column_start),
            position(&offset_encoding, span, span.line_end, span.column_end),
        )
    };
    lsp_types::Location::new(uri, range)
}

fn position(
    offset_encoding: &OffsetEncoding,
    span: &DiagnosticSpan,
    line_offset: usize,
    column_offset: usize,
) -> lsp_types::Position {
    let line_index = line_offset - span.line_start;

    let mut true_column_offset = column_offset;
    if let Some(line) = span.text.get(line_index) {
        if line.text.chars().count() == line.text.len() {
            // all one byte utf-8 char
            return lsp_types::Position {
                line: (line_offset as u32).saturating_sub(1),
                character: (column_offset as u32).saturating_sub(1),
            };
        }
        let mut char_offset = 0;
        let len_func = match offset_encoding {
            OffsetEncoding::Utf8 => char::len_utf8,
            OffsetEncoding::Utf16 => char::len_utf16,
        };
        for c in line.text.chars() {
            char_offset += 1;
            if char_offset > column_offset {
                break;
            }
            true_column_offset += len_func(c) - 1;
        }
    }

    lsp_types::Position {
        line: (line_offset as u32).saturating_sub(1),
        character: (true_column_offset as u32).saturating_sub(1),
    }
}
```