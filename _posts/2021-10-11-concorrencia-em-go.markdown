---
layout: post
title:  "Concorrência em Go"
date:   2021-10-11 17:30:00 -0300
---

# Concorrência em Go

Em Go é possível fazer programas que utilizam de concorrência para melhorar o seu desempenho. A concorrência consiste em executar múltiplas funções independentes ao mesmo tempo para tentar otimizar a utilização de recursos e diminuir o tempo gasto com processamento.

Suponhamos que temos uma API que faz duas requisições para APIs de terceiros e cada uma delas demore 1 segundo. Em um código sequencial sem concorrência, nossa API demoraria no mínimo 2 segundos para responder. Caso as requisições para essas duas APIs sejam independentes, é possível fazer essas chamadas ao mesmo tempo. Essa abordagem poderia diminuir em 50% o tempo de resposta da nossa API somente com essa simples alteração para utilizar concorrência.

**Sem concorrência**

Abaixo vemos a imagem que representa o fluxo sem concorrência onde as requisições para serviços externos são feitas em sequência.

![]({{site.url}}/assets/img/concorrencia-em-go/intro-sem-concorrencia-2.png)

**Código sem concorrência**

```golang
package main

import (
	"log"
	"time"
)

func main() {
	log.Println("inicio")
	log.Println(requestApiTerceiros1())
	log.Println(requestApiTerceiros2())
	log.Println("fim")
}

func requestApiTerceiros1() string {
	time.Sleep(1 * time.Second)
	return "resposta api terceiros 1"
}

func requestApiTerceiros2() string {
	time.Sleep(1 * time.Second)
	return "resposta api terceiros 2"
}
```

**Resultado da execução do código**
```golang
go run main.go
2021/10/11 16:59:35 inicio
2021/10/11 16:59:36 resposta api terceiros 1
2021/10/11 16:59:37 resposta api terceiros 2
2021/10/11 16:59:37 fim
```

**Com concorrência**

Ao utilizar concorrência ambas as requisições podem ser feitas ao mesmo tempo para diminuir o tempo como mostra o fluxo da imagem abaixo.

![]({{site.url}}/assets/img/concorrencia-em-go/intro-com-concorrencia-2.png)

**Código com concorrência**
```golang
package main

import (
	"log"
	"time"
)

func main() {
	log.Println("inicio")
	go requestApiTerceiros1()
	go requestApiTerceiros2()
	time.Sleep(5 * time.Second)
}

func requestApiTerceiros1() {
	time.Sleep(1 * time.Second)
	log.Println("resposta api terceiros 1")
}

func requestApiTerceiros2() {
	time.Sleep(1 * time.Second)
	log.Println("resposta api terceiros 2")
}
```
**Resultado da execução do código**
```golang
go run main2.go
2021/10/11 17:01:17 inicio
2021/10/11 17:01:18 resposta api terceiros 2
2021/10/11 17:01:18 resposta api terceiros 1
```

Em suma, podemos ver que conseguimos otimizar nosso código usando concorrência. Desse modo, é importante saber como trabalhar com concorrência para ter a habilidade de fazer o uso dessa característica da linguagem quando for apropriado. 

É importante conhecer alguns conceitos da linguagem para trabalhar bem com concorrência, são eles:
- Go Routines
- Channels
- WaitGroup

## Go Routines

Uma Go routine é uma thread leve que é gerenciada pelo Go e é utilizada para rodar algum código específico que esteja dentro de uma função.

Não é preciso ter nenhuma característica específica para conseguir executar uma função dentro de uma Go Routine, basta utilizar a sintaxe apropriada durante a chamada dessa função.

### Chamada de função normal

```golang
func main(){
	sayHi(1)
}

func sayHi(n int){
	fmt.Printf("hi %d\n", n)
}
```

### Chamada de função com Go Routine

```golang
func main(){
	go sayHi(1)
}
```

Ao executar ambos os códigos é possível notar que na chamada normal de função é exibido na tela o texto normalmente, enquanto que ao chamar utilizando Go Routine o texto não é exibido. 

O motivo de isso acontecer é porque esse código executa a função sayHi em uma thread separada e não fica esperando a finalização dessa thread. Como não existe mais nenhum código após a criação dessa Go Routine então o programa é finalizado sem nunca ter o retorno da função. Um jeito simples de resolver isso é adicionar um tempo de espera no código da seguinte forma:

```golang
func main(){
	go sayHi(1)
	time.Sleep(1 * time.Second)
}
```

Apesar dessa abordagem resolver o problema, ela está longe de ser a ideal pois nunca saberemos exatamente o tempo necessário para uma função ser executada e dessa forma estamos adicionando atraso no nosso código.

Esse exemplo acima foi feito apenas para entender a sintaxe utilizada para executar funções dentro de Go Routines e mostrar que o código não aguarda a execução dessas funções antes de continuar seu fluxo normal. Para o gerenciamento correto de Go Routines o Go possui mecanismos próprios para isso como **Channel** e **WaitGroup**.

## Channels

Channel é uma maneira que o Go possui de permitir a comunicação entre diferentes partes do código. Um dos cenários possíveis para se utilizar channels é para a comunicação entre Go Routines.

Para criar um novo channel basta criar uma nova variável do tipo `chan`, por exemplo:

```golang
var out chan string // valor default é nil
out := make(chan string, 1) // cria um channel vazio do tipo string que é capaz de armazenar 1 elemento
```

Ao criar uma variável do tipo channel o seu valor default é nil. Para inicializar o channel como sendo um channel vazio é preciso utilizar a função `make`, assim como é feito com variáveis do tipo slice e map.

O operador utilizado para inserir ou ler dados de um channel é o operador `<-`, com o sentido dessa seta apontando para onde o dado está indo. 

**Código**
```golang
out := make(chan string, 1)
out <- "myString" //Adiciona uma string no channel criado
fmt.Println(<-out) //Imprime string armazenada no channel criado
```
**Resultado**
```
myString
```

Um fato interessante sobre como channels funcionam é que ao tentar ler o conteúdo de um channel vazio ou ao tentar escrever um dado em um channel sem espaço o código pára e fica esperando até ter espaço suficiente no channel. Podemos ver esse comportamento ao usar o exemplo acima só que removendo a escrita no channel:

**Código**
```golang
out := make(chan string, 1)
fmt.Println(<-out) //Imprime string armazenada no channel criado
```
**Resultado**
```
fatal error: all goroutines are asleep - deadlock!

goroutine 1 [chan receive]:
main.main()
        /home/breno/programming/tmp/main.go:10 +0x39
exit status 2
```

A mensagem de erro mostra que ocorreu um deadlock pois não existe nenhuma Go Routine sendo executada além da função main e ela está parada esperando algo ser escrito no channel. O deadlock ocorre porque nunca será escrito algo no channel já que o main está parado tentando ler do canal e não existe nenhuma outra Go Routine sendo executada. Se criarmos uma Go Routine que não faz nada e que fique rodando direto, o erro não ocorre mais. Veja o exemplo abaixo:
**Código**
```golang
func main() {
	out := make(chan string, 1)
	go func() {
		for {

		}
	}()
	fmt.Print(<-out)
}
```
**Resultado**
```
Nada impresso na tela
```

Se for parar pra pensar, essa abordagem para solucionar o erro de deadlock nesse exemplo não resolve nenhum problema de fato pois esse problema de deadlock só existe porque estamos usando channel em um local que deveríamos estar usando uma simples variável do tipo string. No entanto, haverá casos onde essa tática será útil  como quando formos construir alguma pipeline para execução de código concorrente.

### Compartilhando channels entre Go Routines

É possível compartilhar um channel entre várias Go Routines que estejam sendo executadas ao mesmo tempo como mostra o exemplo abaixo. Nesse código quatro diferentes rotinas são executadas ao mesmo tempo e a medida que dados são escritos no channel eles são mostrados na tela.

```golang
func main() {
	out := make(chan string)
	go sayHi(1, out)
	go sayHi(2, out)
	go sayHi(3, out)
	go sayHi(4, out)

	for i := 1; i <= 4; i++ {
		fmt.Println(<-out)
	}
}

func sayHi(n int, out chan string) {
	out <- fmt.Sprintf("hi %d", n)
}
```

É interessante notar que o mesmo channel está sendo usado nas quatro Go Routines criadas. O channel foi criado com a função `make` sem passar o tamanho como segundo argumento, isso indica que o channel criado não possui buffer. Na prática isso indica que um dado só poderá fluir nesse canal quando tiver uma parte do código lendo e outra escrevendo pois não existe nenhum local para armazenar esse valor. 

No exemplo acima, assim que uma Go Routine escrever no canal o channel será bloqueado para escrita até esse valor for removido do canal pela função Println, sendo assim não existe uma concorrência de fato nesse código já que as Go Routines criadas estão sempre aguardando a liberação do canal.

Também nesse exemplo, o programa não garante ordem de execução das Go Routines visto que são threads de execução independentes.

## WaitGroup
