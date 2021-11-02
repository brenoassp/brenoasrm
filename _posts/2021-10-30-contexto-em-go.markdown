---
layout: post
title: "Contexto em Go"
date: 2021-10-30 08:00:00 -0300
---

## Qual problema o Contexto resolve?

_Context_ é um pacote em Go desenvolvido para:

1 - Permitir informação com escopo da requisição ser passada para frente durante o ciclo de vida da requisição. Isso permite o acesso a essas informações em várias partes do código, inclusive em funções executadas de maneira concorrente em diferentes Go Routines.

2 - Permitir o cancelamento prematuro de código concorrente quando ocorrer algum erro ou o prazo para o código ser executado expirar.

## Introdução

Um código concorrente possui diversas Go Routines sendo executadas ao mesmo tempo e frequentemente é necessário abortar a execução desse código quando algo dá errado em uma dessas funções, caso contrário recursos serão gastos desnecessariamente.

Uma forma de manter a comunicação entre todas as Go Routines sendo executadas é utilizando contexto. Quando uma requisição inicia, um novo contexto é criado. A partir desse contexto, outros podem ser criados, formando uma árvore de contextos. Nessa árvore, quando um contexto finaliza a execução por algum motivo, todos os contextos filhos também são finalizados.

Outra possibilidade de uso do contexto é armazenar informações com escopo da requisição como o requestID. Desse modo, caso ocorrer erro em algum momento do código, esse ID pode ser utilizado na resposta e/ou no log de erro.

O pacote contexto define um tipo com os seguintes métodos:

```go
type Context interface {
	// Deadline returns the time when work done on behalf of this context
	// should be canceled. Deadline returns ok==false when no deadline is
	// set. Successive calls to Deadline return the same results.
	Deadline() (deadline time.Time, ok bool)

	// Done returns a channel that's closed when work done on behalf of this
	// See https://blog.golang.org/pipelines for more examples of how to use
	// a Done channel for cancellation.
	Done() <-chan struct{}

	// If Done is not yet closed, Err returns nil.
	// If Done is closed, Err returns a non-nil error explaining why:
	// Canceled if the context was canceled
	// or DeadlineExceeded if the context's deadline passed.
	// After Err returns a non-nil error, successive calls to Err return the same error.
	Err() error

	// Value returns the value associated with this context for key, or nil
	// if no value is associated with key. Successive calls to Value with
	// the same key returns the same result.
	//
	Value(key interface{}) interface{}
}
```

Os métodos responsáveis pela comunicação de sinais de cancelamento são:

1 - Deadline()

2 - Done()

3 - Err()

O método responsável por retornar valores armazenados no contexto é o Value().

## Usando contextos para trafegar informações

Não é possível adicionar novos valores a um contexto que já existe e nem alterar um valor já existente. Caso alguma dessas operações sejam necessárias é preciso criar um novo contexto. No exemplo abaixo, `newCtx` é o novo contexto criado com a chave `c` e o valor `20`. Se for necessário adicionar um outro valor no contexto, basta fazer a mesma coisa utilizando o `newCtx` como parâmetro.

```go
func main() {
	http.HandleFunc("/", func(w http.ResponseWriter, req *http.Request) {
		ctx := req.Context()
		newCtx := context.WithValue(ctx, "c", 20)

		fmt.Printf("%v\n", newCtx.Value("c"))

		w.Write([]byte(`{}`))
	})

	log.Fatal(http.ListenAndServe(":8080", nil))
}
```

Apesar de ser possível utilizar contexto para trafegar qualquer tipo de informação, essa prática não é considerada boa e é preciso avaliar com cuidado se há realmente necessidade do tráfego da informação ser através de contexto.

**Cenário adequado para tráfegar informações com contexto:** Se temos um dado na raíz e precisamos dele em várias partes do código para um caso de uso que pode ocorrer ou não como logs, audit, gerenciamento de goroutines, timeouts e etc.

**Cenário não adequado para tráfegar informações com contexto:** Se temos a necessidade de passar alguma informação no código e não esse tipod e informação não se encaixa no cenário acima é inapropriado utilizar o contexto porque o código deixa de receber argumentos explícitos que são usados na lógica da aplicação e passa a receber informações implícitas dentro do contexto. Essa característica deixa o código mais difícil de entender e mais difícil de escrever.

## Usando contextos para abortar execução de código que esteja demorando muito

Vamos ver como ficaria um exemplo de uso de contexto para abortar a execução de código concorrente. Nosso código será uma API com uma rota de `/calcularFrete`. O trabalho dessa rota será verificar de forma concorrente o preço do frete em duas APIs diferentes, a API dos correios e a API da sequoia. Nossa API retornará assim que receber a primeira resposta, independente do valor. Outra característica da nossa API é que essa rota terá um prazo máximo de 2 segundos para resposta, ou seja, caso as APIs dos serviços externos demorarem mais que isso nós abortaremos a execução usando contexto.

O primeiro passo a ser feito é criar um contexto com um prazo de 2 segundos. Ao expirar esse prazo, é enviado um sinal para um channel que pode ser escutado com o métodp Done(). Ou seja, utilizamos esse `channel` para verificar se devemos ou não abortar a execução.

No código foi utilizado `sleep` para simular o tempo de resposta de cada uma das APIs externas.
Para testar os diferentes cenários do código abaixo, basta alterar o tempo de duração dos `sleeps`. Colocando valores maior que 2 para ambas as APIs de calcular frete conseguimos ver que o `timeout` ocorre. Já quando uma delas possui um valor menor que 2 o código consegue executar normalmente.

![]({{site.url}}/assets/img/contexto-go/api-calcular-frete.png)

```go
package main

import (
	"context"
	"fmt"
	"net/http"
	"time"
)

func main() {
	http.HandleFunc("/", calculaFrete)
	http.ListenAndServe(":8080", nil)
}

func getFreteCorreios(ctx context.Context) (float32, error) {
	r := make(chan float32)

	// calcula valor do frete usando os Correios como transportadora
	go requestFreteFor("correios", r)

	select {
	case <-ctx.Done():
		// tempo expirou
		return 0, fmt.Errorf("Timeout")
	case c := <-r:
		// deu certo
		return c, nil
	}
}

func requestFreteFor(transportadora string, result chan float32) {
	switch transportadora {
	case "correios":
		time.Sleep(5 * time.Second) // aguardando valor do frete usando os Correios como transportadora
		result <- 10
	case "sequoia":
		time.Sleep(3 * time.Second) // aguardando valor do frete usando os Correios como transportadora
		result <- 15
	}
}

func getFreteSequoia(ctx context.Context) (float32, error) {
	r := make(chan float32)

	// calcula valor do frete usando a Sequoia como transportadora
	go requestFreteFor("sequoia", r)

	select {
	case <-ctx.Done():
		// tempo expirou
		return 0, fmt.Errorf("Timeout")
	case c := <-r:
		// deu certo
		return c, nil
	}
}

func calculaFrete(w http.ResponseWriter, req *http.Request) {
	ctx, cancel := context.WithTimeout(req.Context(), 2*time.Second)
	defer cancel()

	result := make(chan float32)
	go func() {
		value, err := getFreteCorreios(ctx)
		if err != nil {
			// log de erro
			return
		}
		result <- value
	}()

	go func() {
		value, err := getFreteSequoia(ctx)
		if err != nil {
			// log de erro
			return
		}
		result <- value
	}()

	select {
	case <-ctx.Done():
		fmt.Println("Não terminou a tempo")
	case r := <-result:
		fmt.Printf("Terminou a tempo e o valor do frete calculado mais rapido foi: %.2f\n", r)
	}
}
```
