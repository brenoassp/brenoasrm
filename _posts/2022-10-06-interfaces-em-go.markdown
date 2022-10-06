---
layout: post
title: "Trabalhando com interfaces em Go"
date: 2022-10-06 18:00:00 -0300
---

## Trabalhando com interfaces em Go

Interface é um tipo de dado que define um conjunto de métodos.
Por exemplo, podemos criar uma interface chamada `FormaGeometrica` que todas as formas geométricas deverão implementar pois todas elas possuem um perímetro e uma área.

```go
type FormaGeometrica interface {
    Perimetro() float32
    Area() float32
}
```

Podemos criar, por exemplo, o tipo `Circulo` que implementa essa interface da seguinte forma:

```go
type Circulo struct {
	raio float32
}

func NewCirculo(raio float32) Circulo {
	return Circulo{
		radius: raio,
	}
}

func (c Circulo) Perimetro() float32 {
	return 2 * math.Pi * c.raio
}

func (c Circulo) Area() float32 {
	return math.Pi * c.raio * c.raio
}
```

Note que não tem nenhuma indicação explícita indicando que o tipo `Circulo` implementa a interface `FormaGeometrica`. Isso ocorre porque em Go isso é feito de forma implícita, ou seja, se um tipo implementa os métodos contidos em uma interface, esse tipo automaticamente implementa essa interface.

Como não especificamos explicitamente que um tipo implementa uma interface específica, não é possível gerar um erro de compilação caso o tipo não implemente essa interface, mas isso pode ser resolvido com um truque simples. Podemos adicionar a seguinte linha no nosso código:

```go
var _ FormaGeometrica = NewCirculo(3)
```

Aqui estamos usando o identificador em branco para definir uma variável que nunca será utilizada no código. Como essa variável é do tipo da interface que criamos, teremos um erro de compilação caso a `struct Circulo` não implementar essa interface. Fazendo isso, conseguimos forçar nossos tipos a implementar alguma interface em específica.

Outro ponto importante é saber que existe a `interface{}` que nada mais é do que uma interface que não possui nenhum método, ou seja, todos os tipos de dados implementam essa interface. Isso significa que se algum método receber  um parâmetro do tipo `interface{}`, qualquer variável poderá ser enviado nesse parâmetro, independente do tipo.

Vamos criar agora mais um tipo para exemplificar:

```go
type Quadrado struct {
	lado float32
}

func NewQuadrado(lado float32) Quadrado {
	return Quadrado{
		lado: lado,
	}
}

func (c Quadrado) Perimetro() float32 {
	return 4 * c.lado
}

func (c Quadrado) Area() float32 {
	return c.lado * c.lado
}
```

Agora podemos usar esses tipos em variáveis do tipo interface de `FormaGeometrica`. No código abaixo está sendo criada uma slice do tipo `FormaGeometrica` que pode armazenar variáveis tanto do tipo `Circulo` quanto do tipo `Quadrado`. Ao iterar sobre os elementos dessa slice, não precisamos saber qual o tipo concreto de cada elemento, basta utilizar os métodos definidos pela interface.

```go
package main

import "fmt"

type FormaGeometrica interface {
	Perimetro() float32
	Area() float32
}

func main() {
	var formas []FormaGeometrica
	circulo := NewCirculo(2)
	quadrado := NewQuadrado(2)
	formas = append(formas, circulo)
	formas = append(formas, quadrado)

	for _, f := range formas {
		fmt.Println(f.Perimetro())
	}
}
```