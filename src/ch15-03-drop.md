## A Trait `Drop` Roda Código durante a Limpeza

A segunda trait de importância para a pattern de ponteiros inteligentes é a
`Drop`, que nos permite personalizar o que acontece quando um valor está prestes
a sair de escopo. Nós podemos prover uma implementação da trait `Drop` para
qualquer tipo, e o código que especificarmos pode ser usado para liberar
recursos como arquivos ou conexões de rede. Estamos introduzindo `Drop` no
contexto de ponteiros inteligentes porque a funcionalidade da trait `Drop` é
usada quase sempre quando estamos implementando ponteiros inteligentes. Por
exemplo, o `Box<T>` customiza `Drop` para desalocar o espaço no heap para o qual
o box aponta.

Em algumas linguagens, a pessoa que está programando deve chamar código para
liberar memória ou recursos toda vez que ela termina de usar uma instância de um
ponteiro inteligente. Se ela esquece, o sistema pode ficar sobrecarregado e
falhar. No Rust, podemos especificar que um pedaço específico de código deva ser
rodado sempre que um valor sair de escopo, e o compilador irá inserir esse
código automaticamente. Assim, não precisamos cuidadosamente colocar código de
limpeza em todos os lugares de um programa em que uma instância de um tipo
específico deixa de ser usada, e ainda assim não vazaremos recursos!

Para especificar o código que vai rodar quando um valor sair de escopo, nós
implementamos a trait `Drop`. A trait `Drop` requer que implementemos um método
chamado `drop` que recebe uma referência mutável de `self`. Para ver quando o
Rust chama `drop`, vamos implementar `drop` com declarações de `println!` por
ora.

A Listagem 15-14 mostra uma struct `CustomSmartPointer`
("PonteiroInteligentePersonalizado") cuja única funcionalidade é que ela irá
imprimir `Destruindo CustomSmartPointer!` quando a instância sair de escopo.
Este exemplo demonstra quando o Rust roda a função `drop`:

<span class="filename">Arquivo: src/main.rs</span>

```rust
struct CustomSmartPointer {
    data: String,
}

impl Drop for CustomSmartPointer {
    fn drop(&mut self) {
        println!("Destruindo CustomSmartPointer com dados `{}`!", self.data);
    }
}

fn main() {
    let c = CustomSmartPointer { data: String::from("alocado primeiro") };
    let d = CustomSmartPointer { data: String::from("alocado por último") };
    println!("CustomSmartPointers criados.");
}
```

<span class="caption">Listagem 15-14: Uma struct `CustomSmartPointer` que
implementa a trait `Drop` onde colocaríamos nosso código de limpeza</span>

A trait `Drop` é incluída no prelúdio, então não precisamos importá-la. Nós
implementamos a trait `Drop` no `CustomSmartPointer` e providenciamos uma
implementação para o método `drop` que chama `println!`. O corpo da função
`drop` é onde você colocaria qualquer que fosse a lógica que você gostaria que
rodasse quando uma instância do seu tipo for sair de escopo. Aqui estamos
imprimindo um texto para demonstrar o momento em que o Rust chama `drop`.

Na `main`, nós criamos duas instâncias do `CustomSmartPointer` e então
imprimimos `CustomSmartPointers criados.`. No final da `main`, nossas instâncias
de `CustomSmartPointer` sairão de escopo, e o Rust irá chamar o código que
colocamos no método `drop`, imprimindo nossa mensagem final. Note que não
tivemos que chamar o método `drop` explicitamente.

Quando rodarmos esse programa, veremos a seguinte saída:

```text
CustomSmartPointers criados.
Destruindo CustomSmartPointer com dados `alocado por último`!
Destruindo CustomSmartPointer com dados `alocado primeiro`!
```

O Rust chamou automaticamente `drop` para nós quando nossa instância saiu de
escopo, chamando o código que especificamos. Variáveis são destruídas na ordem
contrária à de criação, então `d` foi destruída antes de `c`. Esse exemplo serve
apenas para lhe dar um guia visual de como o método `drop` funciona, mas
normalmente você especificaria o código de limpeza que o seu tipo precisa rodar
em vez de imprimir uma mensagem.

### Destruindo um Valor Cedo com `std::mem::drop`

Infelizmente, não é simples desabilitar a funcionalidade automática de `drop`.
Desabilitar o `drop` normalmente não é necessário; o ponto todo da trait `Drop`
é que isso seja feito automaticamente. Mas ocasionalmente, você pode querer
limpar um valor cedo. Um exemplo é quando usamos ponteiros inteligentes que
gerenciam locks: você pode querer forçar o método `drop` que libera o lock a
rodar para que outro código no mesmo escopo possa adquiri-lo. O Rust não nos
deixa chamar o método `drop` da trait `Drop` manualmente; em vez disso, temos
que chamar a função `std::mem::drop` disponibilizada pela biblioteca padrão se
queremos forçar um valor a ser destruído antes do fim de seu escopo.

Vamos ver o que acontece quando tentamos chamar o método `drop` da trait `Drop`
manualmente, modificando a função `main` da Listagem 15-14, conforme mostra a
Listagem 15-15:

<span class="filename">Arquivo: src/main.rs</span>

```rust,ignore
fn main() {
    let c = CustomSmartPointer { data: String::from("algum dado") };
    println!("CustomSmartPointer criado.");
    c.drop();
    println!("CustomSmartPointer destruído antes do fim da main.");
}
```

<span class="caption">Listagem 15-15: Tentando chamar o método `drop` da trait `Drop` manualmente para limpar cedo</span>

Quando tentamos compilar esse código, recebemos este erro:

```text
erro[E0040]: uso explícito de método destrutor
  --> src/main.rs:14:7
   |
14 |     c.drop();
   |       ^^^^ chamadas explícitas a destrutores não são permitidas
```

Essa mensagem de erro afirma que não nos é permitido chamar explicitamente
`drop`. A mensagem de erro usa o termo _destrutor_, que é um termo geral de
programação para uma função que limpa uma instância. Um _destrutor_ é análogo a
um _construtor_, que cria uma instância. A função `drop` em Rust é um destrutor
específico.

O Rust não nos deixa chamar `drop` explicitamente porque o `drop` ainda seria
chamado no valor ao final da `main`. Isso seria um erro de _liberação dupla_
(_double free_) porque o Rust estaria tentando limpar o mesmo valor duas vezes.

Nós não podemos desabilitar a inserção automática do `drop` quando um valor sai
de escopo, e também não podemos chamar o método `drop` explicitamente. Então, se
precisamos forçar um valor a ser limpo antes, podemos usar a função
`std::mem::drop`.

A função `std::mem::drop` é diferente do método `drop` na trait `Drop`. Nós a
chamamos passando como argumento o valor que queremos forçar a ser destruído
cedo. Essa função está no prelúdio, então podemos modificar a `main` na Listagem
15-14 para chamar a função `drop`, como mostra a Listagem 15-16:

<span class="filename">Arquivo: src/main.rs</span>

```rust
# struct CustomSmartPointer {
#     data: String,
# }
#
# impl Drop for CustomSmartPointer {
#     fn drop(&mut self) {
#         println!("Destruindo CustomSmartPointer!");
#     }
# }
#
fn main() {
    let c = CustomSmartPointer { data: String::from("algum dado") };
    println!("CustomSmartPointer criado.");
    drop(c);
    println!("CustomSmartPointer destruído antes do final da main.");
}
```

<span class="caption">Listagem 15-16: Chamando `std::mem::drop` para destruir um
valor explicitamente antes que ele saia de escopo</span>

Rodar esse código irá imprimir o seguinte:

```text
CustomSmartPointer criado.
Destruindo CustomSmartPointer com dados `algum dado`!
CustomSmartPointer destruído antes do final da main.
```

O texto `` Destruindo CustomSmartPointer com dados `algum dado`! `` é impresso
entre o texto `CustomSmartPointer criado.` e `CustomSmartPointer destruído antes do final da main.`, mostrando que o método `drop` é chamado para destruir o `c`
naquele ponto.

Podemos usar o código especificado em uma implementação da trait `Drop` de
várias maneiras para tornar a limpeza conveniente e segura: por exemplo,
poderíamos usá-lo para criar nosso próprio alocador de memória! Com a trait
`Drop` e o sistema de posse do Rust, não temos que lembrar de fazer a limpeza
porque o Rust faz isso automaticamente.

Também não temos que nos preocupar em acidentalmente limpar valores ainda em uso
porque isso causaria um erro de compilação: o sistema de posse que garante que
as referências são sempre válidas também garante que o `drop` é chamado apenas
uma vez quando o valor não está mais sendo usado.

Agora que examinamos o `Box<T>` e alguma características de ponteiros
inteligentes, vamos dar uma olhada em alguns outros ponteiros inteligentes
definidos na biblioteca padrão.
