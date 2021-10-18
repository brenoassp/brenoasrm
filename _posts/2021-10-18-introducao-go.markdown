---
layout: post
title: "Introdução à Linguagem Go"
date: 2021-10-18 10:20:00 -0300
---

# Por quê usar Go?

A linguagem Go possui vários pontos positivos como performance, legibilidade e simplicidade,

### Performance

- Eficiência de linguagem com tipagem estática.

- Compilação rápida.

- Suporte a concorrência.

### Legibilidade

Uma única forma de fazer as coisas garante facilidade de leitura e escrita do código. Por exemplo, o Javascript possui a função map, já no Go precisamos utilizar o for para fazer algo similar. Veja abaixo o exemplo comparando a implementação do map com o for no Go e utilizando o map via Javascript.

```javascript
// Javascript
numbers = [4, 9, 16, 25];
result = numbers.map(Math.sqrt);
```

```go
// Go
package main

import (
    "fmt"
    "math"
)

func main() {
    numbers := []int{4, 9, 16, 25}
    result := []float64{}
    for _, value := range numbers {
        result = append(result, math.Sqrt(float64(value)))
    }
    fmt.Println(result)
}
```

A princípio a implementação em Go pode parecer mais complicada por exigir mais linhas de código, mas na prática é mais simples de lidar com o código pelos seguintes motivos:

1 - Por existir apenas essa forma de implementação o desenvolvedor não precisa pensar qual das alternativas disponíveis ele irá utilizar naquele momento. Isso pode parecer algo irrelevante, mas não prática não é.

2 - Ao ter várias features diferentes que podem ser utilizadas para o mesmo propósito, o desenvolvedor necessita saber sobre a implementação interna dessas features para escolher qual delas usar. Da mesma forma, outros desenvolvedores que trabalharão futuramente no código precisarão desse conhecimento e ao lerem o código "perderão tempo" pensando o porquê da implementação ter sido feita daquela forma e não de outra maneira.

3 - Em muito dos casos, o desenvolvedor não terá o conhecimento da estrutura interna da linguagem e acabará usando uma feature com performance pior sem necessidade.

### Simplicidade

Gerenciamento de memória com garbage collector.

Geração de documentação automática com godoc.

## Por quê Go foi criado?

Go foi criado com o objetivo de ter desempenho similar ao de linguagens estaticamente tipadas como C++, mas sem a complexidade envolvida durante o desenvolvimento do código. Nesse sentido, o objetivo era ter a facilidade de escrita de código de uma linguagem dinâmica como Python.

Além disso, Go foi criado com concorrência em mente, diferente de linguagens como C++ e Java que introduziram essa característica ao longo do tempo visto que são linguagens bem mais antigas.

Por fim, os criadores de Go utilizaram a experiência de muitos anos na área de desenvolvimento e todas as duas dores desse período para fazer uma linguagem que resolvesse os problemas existentes de forma elegante e simples.

## Casos de Uso

- CLI
- Scripts
- Desenvolvimento Web
- Serviços

## O que é preciso saber para fazer o primeiro programa em Go

O primeiro passo é instalar o Go que pode ser baixado no site https://golang.org/

**Instalação no Windows**
Para instalar o Go no windows basta baixar o arquivo executável contido no site oficial e seguir o passo a passo de instalação ao abrí-lo. Ao finalizar a instalação o caminho do Go já estará configurado na variável de ambiente e todos os comandos já poderão ser usados. Vale lembrar que é preciso reabrir janelas do prompt para a configuração surtir efeito.

**Instalação no Linux**
No caso do Linux a instalação nada mais é do que baixar um arquivo compactado que contém uma pasta com todos os arquivos necessários para o Go funcionar. Com esse arquivo em mãos, basta descompactá-lo num diretório específico do sistema como o `/user/local/go` e adicionar na variável de ambiente PATH o caminho da pasta que contém os binários do Go. Em resumo o passo a passo é o seguinte (considerando a versão do Go 1.17.2):

```bash
# Descompactando o arquivo
tar -C /usr/local -xzf go1.17.2.linux-amd64.tar.gz

# adicionar no ~/.bashrc ou ~/.zshrc ou qualquer outro arquivo de configuração do shell que você esteja usando
export PATH=$PATH:/usr/local/go/bin
```

Após a instalação bem sucedida basta verificar se tudo está ok executando o comando abaixo para verificar se a versão instalada será mostrada.

```bash
go version
```

Agora que temos o Go instalado já conseguimos compilar e executar códigos em Go. Crie um arquivo com nome **main.go** com o seguinte conteúdo:

```go
package main

import (
	"fmt"
)

func main() {
	fmt.Println("Hello World!")
}
```

Esse programa imprime na tela as palavras `Hello World` utilizando para isso a função Println do pacote `fmt` que faz parte das bibliotecas padrão do Go.

- fmt é o nome do pacote. Pacotes possuem todos os caracteres minúsculos.
- Println é o nome da função e nesse caso ela começa com letra maiúscula e isso não ocorre por acaso. Todas as partes públicas dentro de um pacote, ou seja, que podem ser acessadas de fora desse pacote, devem iniciar com letra maíuscula.

Em seguida basta executar o código com o comando:

```bash
go run main.go
```

Lembrando que Go é uma linguagem compilada, o comando `go run` compila e executa o programa feito. Caso queira criar um binário desse programa basta utilizar o `go build`, por exemplo:

```bash
go build main.go
```

## Estruturas básicas da linguagem

### Declarando variáveis

#### Declarando variáveis sem inicializá-las

```go
var x string // valor default string vazia ""
var y int // valor default 0
var z []string // valor default nil
```

#### Declarando variáveis e inicializando ao mesmo tempo

```go
x := "MinhaString"
y := 10
z := []string{"s1","s2","s3"}
```

### Loop

```go
for {
	// loop infinito
}

x := 0
for x < 10 {
	x += 1
}

for i := 0; i < 10; i++ {

}
```

### Structs

```go
type Person struct {
	Name string
	Age int
}
```

### Estrutura condicional

```go
if x < 10 {
	fmt.Println(x)
}
```

### Funções

```go
func add(x int, y int) int {
	return x + y
}
```

## Criando uma API simples

Diferentemente de outras linguagens como PHP, Java e Python, o Go não precisa de um servidor web como Apache ou Nginx para funcionar. O servidor é criado pelo próprio código do Go como mostrado no exemplo abaixo. Esse código é funcional e responde à requisições na url `localhost:8080/ping`.

```go
package main

import (
	"fmt"
	"net/http"
)

func main() {
	http.HandleFunc("/ping", ping)
	http.ListenAndServe(":8080", nil)
}

func ping(w http.ResponseWriter, req *http.Request) {
	fmt.Fprintf(w, "pong\n")
}
```
