# ABI — Application Binary Interface

**ABI (Application Binary Interface)** é o contrato de **baixo nível** que permite código compilado por um compilador, ou escrito em uma linguagem, interagir com código produzido por outro. Enquanto API é acordo em nível de linguagem (headers, nomes de função, tipos abstratos), ABI é o acordo em nível de **máquina**: como argumentos passam, em quais registradores, como a stack fica alinhada, como structs são empacotadas, como símbolos são nomeados, como exceções se propagam.

Se ABI quebra, binário velho para de linkar/rodar contra biblioteca nova — mesmo que a API fonte seja idêntica.

## ISA vs ABI vs API

Três camadas frequentemente confundidas:

| | Define | Consumidor | Exemplo |
|---|---|---|---|
| **ISA** | instruções disponíveis, semântica, registradores | CPU | x86-64, ARMv8, RISC-V RV64GC |
| **ABI** | convenções binárias sobre ISA + OS | linker, loader, compilador | System V AMD64, Microsoft x64, AAPCS |
| **API** | funções, tipos, semântica em source level | programador | POSIX `read()`, Win32 `CreateFileW()` |

Uma ISA pode ter **múltiplos** ABIs (x86-64 tem System V em Linux/Mac e Microsoft x64 em Windows — mesmas instruções, convenções diferentes). Um ABI pode ser implementado em **múltiplas** APIs de linguagem (qualquer linguagem que gere código x86-64 SysV pode interoperar).

## Calling Conventions

O componente mais conhecido do ABI: **como uma função chama outra**.

### Elementos

1. **Passagem de argumentos**: quais em registradores, quais na stack.
2. **Valor de retorno**: onde é colocado.
3. **Caller-saved vs callee-saved**: quem preserva cada registrador.
4. **Stack alignment** na chamada.
5. **Red zone** (ou sua ausência).
6. **Limpeza de stack**: chamador ou chamado remove argumentos.
7. **Tratamento de variadics** (`...`).
8. **Valor de retorno grande** (struct): via registradores múltiplos ou ponteiro oculto.

### x86-64 System V (Linux, macOS, BSD)

Argumentos inteiros/ponteiros nos registradores na ordem:

```
rdi, rsi, rdx, rcx, r8, r9  (primeiros 6)
[stack]                      (7º em diante)
```

Argumentos de ponto flutuante em `xmm0..xmm7`.

- **Retorno**: inteiro em `rax` (+ `rdx` se 128-bit). FP em `xmm0`.
- **Callee-saved**: `rbx, rbp, r12-r15, rsp`.
- **Caller-saved** (voláteis): `rax, rcx, rdx, rsi, rdi, r8-r11`, todos XMM.
- **Stack alignment**: 16 bytes antes do `call` (que empilha RIP, deixando rsp alinhado a 16 após prolog).
- **Red zone**: 128 bytes abaixo de `rsp` utilizáveis sem decrementar. Leaf functions economizam frame.
- **Limpeza**: caller remove argumentos stack (`cdecl`-like).

### Microsoft x64 (Windows)

Argumentos nos registradores:

```
rcx, rdx, r8, r9  (primeiros 4)
[stack]           (5º em diante)
```

FP no `xmm0..xmm3` correspondentes (posição, não tipo — FP e int não se misturam separados).

- **Shadow space**: chamador **reserva 32 bytes** na stack antes de chamar, mesmo se os 4 argumentos estão em registradores. Chamado pode spillar aí.
- **Retorno**: `rax` / `xmm0`.
- **Callee-saved**: `rbx, rbp, rdi, rsi, rsp, r12-r15, xmm6-xmm15` (mais callee-saved que SysV).
- **Stack alignment**: 16 bytes.
- **Sem red zone**.
- **SEH** (Structured Exception Handling) integrado via tabelas.

### x86 (32-bit) — convenções múltiplas

Mundo variado; fonte de bugs interop histórica:

- **cdecl**: argumentos na stack, caller cleanup. Default em GCC, variadics.
- **stdcall**: stack, **callee cleanup**. Win32 API (`WINAPI`).
- **fastcall**: primeiros 2 em `ecx, edx`, resto stack. Variações MS vs Borland.
- **thiscall**: `this` em `ecx` (C++ Microsoft), ou primeiro argumento stack (GCC).
- **pascal, safecall, naked**: históricas.

Portabilidade x86 era pesadelo. x86-64 normalizou em duas principais.

### ARM AAPCS64 (ARMv8-A)

Argumentos inteiros/ponteiros em:

```
x0..x7     (primeiros 8)
[stack]    (8º em diante)
```

FP/SIMD em `v0..v7`.

- **Retorno**: `x0` (+ `x1`), `v0`.
- **Callee-saved**: `x19..x29, sp`.
- **Stack alignment**: 16 bytes sempre.
- **FP (frame pointer)**: `x29`; **LR (link register)**: `x30`.
- **PAC (Pointer Authentication)**: ARMv8.3+ — assinatura de ponteiros anti-ROP via chave em SP. Introduz instruções PACxx/AUTxx no prolog/epilog.

### RISC-V calling convention

Argumentos em `a0..a7` (mapeados a `x10..x17`). FP em `fa0..fa7`.

- Retorno: `a0, a1`.
- Callee-saved: `s0..s11` (`x8, x9, x18..x27`).
- `sp, gp, tp` reservados.
- Alignment 16 em psABI 64-bit.

### Variadic functions

`printf(const char *fmt, ...)` — ABI tem regras específicas:

- **SysV x86-64**: `al` carrega **número de argumentos FP em XMM** (hint ao callee de quantos XMMs foram usados).
- **MS x64**: FP usam tanto XMM quanto registrador inteiro (floats também em `rcx, rdx, r8, r9` duplicado).
- **AAPCS64**: variadics passam tudo em registradores inteiros (FP promovido).

Divergências entre plataformas = origem de bugs em código portável com `va_arg`.

## Nomes de Símbolos e Mangling

Linkers operam em strings. Compiladores geram strings a partir de assinaturas de função.

### C

Símbolo = nome da função. `printf` em libc = `printf`. Às vezes com prefixo underscore (macOS, Windows x86 legado), sem (Linux moderno).

### C++

Overloading + namespaces + templates → exige **name mangling**. Dois esquemas principais:

- **Itanium ABI** (usado por GCC, Clang em Linux/Mac e Windows mingw): prefixos `_Z`. Ex.: `void foo(int, double)` → `_Z3fooid`.
- **MSVC mangling**: formato diferente. `?foo@@YAXHN@Z`.

**Demanglers**: `c++filt` (binutils), `undname.exe` (MSVC), Ghidra embutido.

`extern "C"` em C++ desabilita mangling — nome exposto como C puro. Como FFI C++↔outras linguagens funciona.

### Rust

Mangling próprio, baseado em Itanium v0 desde 2021 (nightly antes). Ex.: `_ZN5alloc5alloc8box_free17h...E`. **Não estável entre versões de compilador** — motivo pelo qual ABI estável em Rust ainda é tópico quente.

### Swift

Mangling riquíssimo codificando tipos genéricos, ARC, protocolos. Swift ABI stable desde 5.0 (2019) em Apple platforms.

### Go

Symbol names específicos; Go binary é mundo próprio. CGo é a ponte via C ABI.

## Struct Layout, Alignment, Padding

Como dados cabem na memória.

### Alignment

Cada tipo tem **alignment requirement**. Ex.: `int32_t` é 4-byte aligned; `double` é 8; SIMD 16. Acesso desalinhado:

- **x86/x64**: lento (e não permitido para SSE movaps).
- **ARM**: historicamente fault; ARMv7/v8 permitem lax mas performance cai.

### Padding em structs

```c
struct Foo {
  char  a;      // offset 0
  // 3 bytes de padding
  int   b;      // offset 4
  char  c;      // offset 8
  // 7 bytes de padding
  double d;     // offset 16
};
// sizeof(Foo) == 24 (não 1+4+1+8=14)
```

Compilador insere padding para respeitar alignment. Reordenar campos economiza espaço:

```c
struct Bar {
  double d;
  int    b;
  char   a;
  char   c;
  // 2 bytes de padding no fim
};
// sizeof(Bar) == 16
```

Regras detalhadas em:

- **C/C++**: implementation-defined, mas psABI fixa para plataforma.
- **Rust**: `#[repr(Rust)]` (default) pode reordenar; `#[repr(C)]` força layout tipo-C; `#[repr(packed)]` elimina padding.
- **Go**: sem reordenamento; programador controla.

### Packing

Pragmas para forçar sem padding:

- **GCC/Clang**: `__attribute__((packed))`.
- **MSVC**: `#pragma pack(1)`.

Custo: acesso desalinhado, lento ou fault. Usado em protocolos de rede, FFI com hardware.

### Bitfields

Ordem de bits em byte depende de endianness e compilador. Notoriamente não-portáveis entre ABIs.

### Endianness

- **Little-endian**: x86, x86-64, ARM (default moderno). Byte menos significativo primeiro.
- **Big-endian**: PowerPC (default), SPARC, "network byte order".
- **Bi-endian**: ARM, MIPS, RISC-V podem ser configurados.

Serialização cross-platform exige especificar endianness (`htonl`, `ntohs`, ou formato explícito).

## Syscall ABI

Interface user ↔ kernel. **Diferente** da calling convention de função normal.

### Linux x86-64

- Instrução: `syscall` (instrução nativa).
- Número da syscall em `rax`.
- Argumentos em `rdi, rsi, rdx, r10, r8, r9` (**`r10` em vez de `rcx`** porque `syscall` clobbera `rcx`).
- Retorno em `rax`. Erros: valor negativo (-errno).
- Números definidos em `<asm/unistd.h>`; estáveis por décadas.

### Linux ARM64

- Instrução: `svc #0`.
- Número em `x8`.
- Argumentos em `x0..x5`.
- Retorno em `x0`.

### Linux 32-bit (x86)

- Historicamente `int 0x80` (lento). Depois `sysenter` via `__kernel_vsyscall`.
- Argumentos em `ebx, ecx, edx, esi, edi, ebp`.

### Windows

Tem syscalls mas são **consideradas privadas** e números **mudam entre versões**. API estável é Win32 (em `kernel32.dll`, `user32.dll`, etc.) que faz a chamada via `ntdll.dll`. Em NT API: `syscall` com número em `eax`, arguments em `rcx, rdx, r8, r9, stack`.

Malware e debuggers que usam números diretos quebram em updates. Supervisors anti-malware às vezes sabem disso.

### macOS

Syscall numbers tem prefixo de **classe** em bits altos. Instrução `syscall`. Preferência é ir via libSystem, não direto.

### BSDs

Parecido com macOS em estrutura. Syscall numbers estáveis dentro de família.

### vDSO e syscall shortcuts

Ver `OPERATING_SYSTEMS.md`. `gettimeofday`, `clock_gettime` rodam no userspace via página mapeada do kernel, sem entrar em kernel mode.

## ABI Stability e Versionamento

### Breaking vs non-breaking

**Source-compatible** != **binary-compatible**. Mudanças em C header que **quebram ABI** mas não source:

- Adicionar campo em struct pública (layout muda, binário antigo não alinha).
- Mudar tamanho de `enum` (ex.: compilador pode usar `int` ou menor).
- Reordenar enum values.
- Adicionar virtual method em classe C++ (vtable muda).
- Mudar inheritance (vtable).
- Mudar template signature.
- Remover ou renomear símbolo exportado.

Library maintainers planejam release ABI-breaking em major bumps (SemVer).

### Opaque types

Técnica para preservar ABI: expor só ponteiros; struct definida em `.c`:

```c
// header público
typedef struct Connection Connection;
Connection* conn_open(const char *url);
void conn_close(Connection*);

// implementação — livre para mudar layout
struct Connection { int fd; char *buf; /* ... */ };
```

Clientes não sabem layout → mudanças internas não quebram ABI. Dominante em C well-designed libraries (SQLite, libcurl, Lua).

### Symbol versioning (ELF)

glibc usa símbolos versionados. `memcpy@GLIBC_2.0`, `memcpy@GLIBC_2.14`. Old binaries ligam a versão antiga; recompiled a nova. Uma libc serve várias gerações. Ferramentas: `readelf -W -a`, `nm -D --with-symbol-versions`.

Linux `--wrap` e `.symver` pragma em código.

### Windows: SxS e manifests

Side-by-Side assembly — múltiplas versões de DLL podem coexistir. Manifest identifica qual app quer qual. Complexo operacionalmente ("DLL Hell" legado).

### Apple framework versioning

Symbols com `__attribute__((availability(...)))`. Weak linking conforme target.

### Abi-check e detection

Ferramentas:

- **abi-compliance-checker, abi-dumper** (Linux open source).
- **AbiView** MSVC.
- **cargo-semver-checks** em Rust.
- Integradas a CI em libs grandes (Qt, GTK, Mesa).

## FFI — Cruzando ABIs de Linguagens

**C ABI é lingua franca**. Quase toda linguagem fala C ABI (extern "C", ctypes, cgo, JNI, etc.).

### Padrões por linguagem

- **Python**: `ctypes`, `cffi`, CPython C API, PyO3 (Rust).
- **JavaScript/Node**: `node-ffi-napi`, `node-addon-api`, N-API.
- **Java**: JNI, Project Panama (FFM API, JEP 442, 454 em Java 22+).
- **Go**: `cgo`. Caros context switches entre runtimes.
- **Rust**: `extern "C"`, crates `libc`, `cc`, `bindgen`, `cbindgen`.
- **Swift**: C interop nativo; Swift ↔ C++ interop desde 5.9.
- **Kotlin/Native, Dart/Flutter**: mecanismos próprios.
- **.NET**: P/Invoke, C++/CLI.
- **WebAssembly**: **Component Model** (2023+) formaliza ABI entre módulos e linguagens em WASM.

### Armadilhas comuns

1. **Layout de struct diferente**: `#[repr(C)]` em Rust obrigatório para interop.
2. **Ownership de memória**: quem aloca, quem libera? Contrato explícito necessário.
3. **Strings**: null-terminated C vs length-prefixed Rust/Go. Conversão custa.
4. **Erros**: exceções C++ **não devem** cruzar ABI boundaries. Retornar código.
5. **Thread safety**: runtime diferente (goroutines + callbacks JNI, etc.).
6. **Calling convention mismatches** em Windows — `CALLBACK`, `WINAPI` macros.
7. **Variadics** raramente funcionam bem cross-language.

## Stack Unwinding e Exceções

Quando exceção lançada (ou `panic` Rust, `abort` em erro), runtime "desenrola" a stack, chamando destructors no caminho.

### DWARF CFI (Call Frame Information)

Padrão em Linux/macOS. Tabelas no ELF (`.eh_frame`) descrevem como restaurar registradores em cada endereço. **Zero-cost** em happy path; custo grande no unwind.

### SEH (Structured Exception Handling)

Windows. Tabelas também, mas modelo diferente — filters permitem inspect e continue.

### setjmp/longjmp

C não tem exceções; `setjmp/longjmp` é primitivo de jump não-local. Não chama destructors. Uso especializado.

### Exceções cruzando ABIs

**Jamais** deixar exceção C++ ou panic Rust atravessar fronteira `extern "C"` para código que não espera — UB. Captura sempre na borda.

## TLS — Thread-Local Storage

Variáveis com instância por thread. ABI para TLS:

- **Linux glibc**: `__thread` (C) ou `thread_local` (C++11); implementado via GOT-relative + `fs:` segmento em x86-64.
- **Windows**: FS/GS segment selects TEB (Thread Environment Block); TLS slots em `TlsAlloc`.
- **macOS**: `fs:` similar; Grand Central Dispatch integra.

Performance cuidadosa — `thread_local` pode ser caro dependendo de modelo (dynamic vs static).

## Exemplos Práticos de ABI Importando

### Glibc vs musl

Mesma linguagem (C), APIs próximas (POSIX), **ABIs incompatíveis**. Binário linkado contra glibc não roda em Alpine (musl). "Static linking" contorna mas inflaciona binário.

### Apple Silicon transition

ARM64 vs x86-64. Rosetta 2 traduz x86-64 → ARM64 em runtime, incluindo emulação de SysV ABI (muitos apps funcionam). Mas não perfeito — AVX-512, memory ordering TSO vs weak.

### Android NDK

Multiple ABIs por arquitetura: armeabi-v7a, arm64-v8a, x86, x86_64. APK carrega múltiplas libs nativas, Android escolhe na instalação.

### Linux x32 ABI

Curiosidade: x86-64 com ponteiros de 32 bits. Menos memória em dado, instruções x64. Pouco usado; quase aposentado.

## Debugging ABI Issues

Sintomas comuns:

- **Segfault em função correta**: argumento em registrador errado → indefinido.
- **Corrupção de stack** em retorno: callee-saved não preservado.
- **Garbled string** em FFI: encoding ou length mismatch.
- **Crash cross-DLL** em Windows: DLL compilada com `/MT` chama `free` em memória alocada em DLL `/MD` — heaps diferentes.

Ferramentas:

- **`objdump -d`, `llvm-objdump`**: disassembly verifica calling convention.
- **GDB** com `layout asm` + `stepi`.
- **Compiler explorer** (godbolt.org): compara codegen entre compiladores/ABIs.
- **strace, ltrace**: vê syscalls e chamadas dinâmicas.
- **ABI check tools** citadas acima.

## Standards e Documentação

- **System V AMD64 psABI** (psabi-x86-64.pdf — mantido em GitLab).
- **ARM AAPCS64** (ARM developer).
- **Microsoft x64 calling convention** (docs.microsoft.com).
- **Itanium C++ ABI** (itanium-cxx-abi/cxx-abi).
- **Linux kernel syscall ABI** — man 2, `arch/x86/entry/syscalls/syscall_64.tbl`.
- **RISC-V psABI** (github.com/riscv-non-isa/riscv-elf-psabi-doc).

psABI = Processor-Specific Application Binary Interface. Documento canônico.

## Recursos

- **"Linkers and Loaders"** — John Levine. Clássico sobre o tema; ABI central.
- **"Operating Systems: Three Easy Pieces"** — seções sobre syscalls.
- **Agner Fog's "Calling conventions for different C++ compilers and operating systems"** (agner.org).
- **psABI specs** (por arquitetura).
- **Itanium C++ ABI doc**: leitura árida mas definitiva.
- **Raymond Chen's blog** (The Old New Thing): histórias sobre ABI Windows.
- **Compiler Explorer (godbolt.org)**: laboratório para comparar codegen.

## Princípios

1. **ABI é contrato binário**; API é contrato textual. Não os confunda.
2. **C ABI é lingua franca** de interop. Quando em dúvida, `extern "C"`.
3. **Calling conventions diferem entre OS na mesma ISA**. Windows x64 ≠ Linux x86-64.
4. **`#[repr(C)]` e `extern "C"`** são os cinto-de-segurança em FFI.
5. **Stable ABI é investimento caro**. Libs maduras usam opaque types e symbol versioning por razão.
6. **Não atravesse ABIs com exceções C++ / panics Rust** sem captura explícita.
7. **Layout de struct é o demônio silencioso**. Padding, alignment, endianness desalinham interop rápido.
8. **Syscall ABI é contrato OS ↔ userspace**. Estável em Linux, volátil em Windows, preferir libc sempre.
9. **Ferramentas de disassembly são amigas** — objdump/godbolt revelam o ABI real.
10. **Teste em múltiplas plataformas** se seu código é portável — ABI issues frequentemente só aparecem fora do laptop do dev.
