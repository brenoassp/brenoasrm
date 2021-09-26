---
layout: post
title:  "Gerenciamento de dependência em Go (Go Modules)"
date:   2021-09-26 09:44:00 -0300
---

# Go Modules

Cada linguagem de programação possui uma maneira de instalar dependências/bibliotecas e gerenciar as versões de cada uma dessas dependências no projeto. A maneira com que o Go gerencia suas dependências desde a versão 1.11 é utilizando Módulos *(Go Modules)*. Em versões passadas do Go as dependências eram gerenciadas com ferramentas auxiliares como o [dep](https://github.com/golang/dep) e é possível migrar para o Go Modules, mais informações sobre como fazer isso pode ser vistas na [wiki do Go](https://github.com/golang/go/wiki/Modules).

Basicamente ao iniciar um novo projeto em Go utilizando na raíz do projeto o comando `go mod init meu-modulo` é criado um novo módulo. Nesse comando o `meu-modulo` pode ter o caminho completo para um repositório de um sistema de controle de versionamento, por exemplo , `github.com/brenoassp/meu-modulo`.

O comando `go mod init` gera um arquivo com nome`go.mod`que contém o nome do módulo criado junto com a versão do Go utilizada, como mostrado abaixo:

```
module github.com/brenoassp/meu-modulo

go 1.17
```

A medida que novos pacotes forem sendo adicionados no projeto esse arquivo vai sendo modificado com os pacotes instalados, por exemplo, podemos instalar um novo pacote com o comando a seguir:

`go get -u github.com/valyala/fasthttp` 

Após isso o arquivo `go.mod` terá o seguinte conteúdo:

```
module github.com/brenoassp/meu-modulo

go 1.17

require (
	github.com/andybalholm/brotli v1.0.3 // indirect
	github.com/klauspost/compress v1.13.6 // indirect
	github.com/valyala/bytebufferpool v1.0.0 // indirect
	github.com/valyala/fasthttp v1.30.0 // indirect
)
```

Apesar de ter instalado apenas uma dependência, mais de um pacote foi instalado, isso acontece porque geralmente todo pacote que instalamos depende também de outros pacotes. Como esses pacotes internos das dependências que instalamos não são utilizados diretamente por nós eles são marcados como dependência indireta com o comentário `indirect` depois da versão. Nesse caso específico, até mesmo a dependência do fasthttp que instalamos está sendo marcada como dependência indireta pois não estamos usando ela em nenhum lugar no código, apesar de ter sido instalada.

Ao criar um código que utilize o fasthttp como o código abaixo, ele deixará de ser uma dependência indireta no nosso projeto.

```
package main

import (
	"fmt"

	"github.com/valyala/fasthttp"
)

func main() {
	fasthttp.ListenAndServe(":8080", func(ctx *fasthttp.RequestCtx) {
		fmt.Fprintln(ctx, "Hello, world")
	})
}
```

O comando `go mod tidy` é responsável por atualizar o `go.mod` seja atualizando ou removendo dependências. Ao executar esse comando o `go.mod`passa a ter o seguinte conteúdo:

```
module github.com/brenoassp/meu-modulo

go 1.17

require github.com/valyala/fasthttp v1.30.0

require (
	github.com/andybalholm/brotli v1.0.3 // indirect
	github.com/klauspost/compress v1.13.6 // indirect
	github.com/valyala/bytebufferpool v1.0.0 // indirect
)

```

Ou seja, o fasthttp deixa de ser uma dependência indireta para ser uma dependência direta já que está sendo utilizado. Se removermos o código que utiliza o fasthttp e executarmos o `go mod tidy` novamente ele removerá do `go.mod` as dependências pois elas não estão sendo utilizadas em nenhum momento.

Além do `go.mod` existe também o arquivo `go.sum` que contém todas as dependências junto com a hash que representa o conteúdo do pacote. Isso serve pra garantir que os pacotes baixados futuramente no projeto são iguais aos pacotes baixados no momento em que a dependência foi adicionada no projeto. Ambos os arquivos devem ser versionados e é recomendável usar o comando `go mod tidy` regularmente para manter esses arquivos atualizados somente com as dependências que estão realmente sendo utilizadas.

Não é necessário ficar utilizando o `go get` para adicionar todos os pacotes que serão utilizados pois ao compilar o projeto (`go build`) ou executar os testes (`go test`) as dependências serão baixadas automaticamente com base nas importações feitas no início do aquivo com extensão `.go`. A única alteração que não é feita automaticamente nesses casos é a verificação se alguma dependência existente no arquivo `go.mod` e `go.sum` já não é mais necessária por não estar sendo utilizada e, consequentemente, não é removida. Nesse caso é preciso executar um `go mod tidy` que removerá esses pacotes não utilizados.

Diferentemente de vários gerenciadores de dependências de outras linguagens, o Go Module não instala as dependências do projeto em uma pasta contida na raiz do repositório. O que controla onde será salvo as bibliotecas são as variáveis de ambiente **GOPATH** e **GOBIN**. Caso essas variáveis não existirem, os pacotes são salvos em `/home/[user]/go`.

**GOBIN:** indica onde serão instalados os binários.
**GOPATH**: indica onde serão instalados os pacotes. Se essa variável existir os binários são instalados na pasta `bin` do primeiro diretório da lista de diretórios existentes nessa variável.

### TL;DR;

|Comando|Exemplo|Descrição
|---|---|---
|go mod init \[path\]| go mod init github.com/brenoassp/meu-modulo | Inicializa um novo módulo no diretório corrente criando um arquivo `go.mod` nesse local
|go mod tidy| - | Atualiza o `go.mod` seja adicionando dependências que estão faltando ou removendo as que não estão sendo utilizadas
|go get modulo|go get https://github.com/valyala/fasthttp|Baixa ou atualiza o pacote fasthttp na versão mais atualizada
|go get -u modulo|go get -u https://github.com/valyala/fasthttp|Atualiza tanto o pacote fasthttp e as suas dependências para a versão mais atualizada
|go mod download| - | Baixa para o cache local todos os módulos existentes no arquivo `go.mod`
