---
layout: post
title: "Construindo um CRUD em Go com persistência em Arquivos"
date: 2021-12-19 14:00:00 -0300
---

## Construindo um CRUD em Go com persistência em Arquivos

### Introdução

O objetivo desse artigo é mostrar o passo a passo necessário para construir uma API de um CRUD (Create, Read, Update, Delete). Ou seja, nossa API permitirá criar, ler, atualizar e deletar pessoas.

Antes de começar de fato a fazer o código, vou mostrar aqui de maneira visual o que pretendo ter contruído no final desse post.

![]({{site.url}}/assets/img/crud-persistencia-arquivo/crud-persistencia-arquivo.png)

A gente pode ver na figura que teremos 4 rotas:

- Rota de POST para criação de pessoas.
- Rota GET para leitura das pessoas. No caso podemos ler todas as pessoas armazenadas ou apenas uma pessoa com base no ID fornecido na URL.
- Rota PUT para atualização de uma pessoa já existente.
- Rota DELETE para remover uma pessoa.

Todas essas rotas vão utilizar a mesma fonte de dados que será um arquivo de nome `person.json`, ou seja, nessa API nós não vamos utilizar banco de dados como forma de armazenamento. Futuramente pretendo fazer um post onde eu modifico esse código atual para utilizar banco de dados ao invés de arquivo, mas nesse caso aqui vou fazer com arquivos para mostrar algumas funções de manipulação de arquivo com Go.

Com isso em mente, a gente já consegue ir para a parte de implementação do código.

### Rota de criação de pessoas (POST /person/)

O primeiro passo é criar o projeto. Para isso, vou abrir um terminal, criar uma pasta com o nome do projeto, no nosso caso vai ser api-crud-persistencia-arquivo. Em seguida, vou inicializar um novo módulo nessa pasta que a gente acabou de criar utilizando o `go mod init github.com/brenoassp/api-crud-persistencia-arquivo`. Agora que nosso módulo já está inicializado, vamos criar aqui um arquivo `main.go` que vai ser a nossa API.

```go
package main

func main(){
    http.HandleFunc("/person/", func(w http.ResponseWriter, r *http.Request){
        http.Error(w, "Not implemented", http.StatusInternalServerError)
    })

    http.ListenAndServe(":8080", nil)
}
```

A primeira coisa que estamos fazendo aqui é criando uma função que será executada quando chegar requisições que começam com `/person/`. Note que nesse caso aqui não estamos escolhendo qual o método HTTP utilizado, ou seja, todas as requisições que se encaixarem no padrão de URL executarão essa mesma função.

Podemos executar o código para ver que independente do método, o retorno será o mesmo. Para isso faça:

`go run main.go`

`curl -XPOST localhost:8080/person/`

`curl -XGET localhost:8080/person/`

Ambos os resultados são iguais pois não há distinção de método e teremos que ter isso em mente durante a implementação da API.

Vou começar primeiramente com a implementação da operação de criação de uma pessoa.

```go
package main

func main(){
    http.HandleFunc("/person/", func(w http.ResponseWriter, r *http.Request){
        if r.Method == "POST" {
            type Person struct {
                ID   int    `json:"id"`
                Name string `json:"name"`
                Age  int    `json:"age"`
            }
            var person Person
            err := json.NewDecoder(r.Body).Decode(&person)
            if err != nil {
                fmt.Printf("Error trying to decode body. Body should be a json. Error: %s\n", err.Error())
                http.Error(w, "Error trying to create person", http.StatusBadRequest)
                return
            }
            if person.ID <= 0 {
                http.Error(w, "person ID should be a positive integer", http.StatusBadRequest)
                return
            }

            // criar pessoa

            // montar resposta
            w.WriteHeader(http.StatusCreated)
            return
        }
        http.Error(w, "Not implemented", http.StatusInternalServerError)
    })

    http.ListenAndServe(":8080", nil)
}
```

Primeiro eu vou verificar se o método da requisição é POST, se sim então podemos continuar.
A primeira operação que devemos fazer é decodificar o corpo da requisição, que no nosso caso deverá ser um JSON com os campos `ID`, `Name` e `Age`. Para isso é preciso criar uma variável de uma struct nesse formato para receber o resultado da decodificação. Com essa variável criada, basta decodificar o JSON recebido da seguinte forma: `json.NewDecoder(r.Body).Decode(&person)`. Esse código tá criando um decodificador com base no corpo da requisição e chamando a função de decodificar passando o endereço de memória da variável que foi criada acima do tipo Person. Esse método retorna um erro que se for diferente de `nil` significa que algo deu errado durante a decodificação. Faremos um tratamento usando esse erro para retornar um erro para o usuário com o status de BadRequest que indica que o erro ocorreu por conta de valor inválido fornecido pelo próprio usuário.

Caso dê tudo certo a resposta será o status http OK (200).

`curl -XPOST localhost:8080/person/ -d '{"id": 1, "name": "joao}'` **(Faltando fechar as aspas duplas no nome, dá erro na decodificação do JSON)**

`curl -XPOST localhost:8080/person/ -d '{"id": 1, "name": "joao"}'` **(Tudo ok)**

Se não tiver dado erro nenhum na decodificação, vou verificar se o ID fornecido é maior do que zero, pois o ID tem que ser positivo.

Se deu tudo certo, agora vou tentar adicionar essa pessoa, persistindo em um arquivo.
A princípio vou só deixar um comentário aqui indicando que essa parte da implementação tá faltando e retornar uma resposta com status de criado.

Antes de implementar essa etapa final da rota, vou extrair a struct que foi criada aqui que representa a pessoa para um arquivo separado. Vou criar uma pasta `domain` e dentro dela o arquivo `entities.go` que terá as entidades. No nosso caso aqui, vou mover a struct Person pra esse arquivo.

```go
// Arquivo: api-crud-salvando-arquivo/domain/entities.go
package domain

type Person struct {
	ID   int    `json:"id"`
	Name string `json:"name"`
	Age  int    `json:"age"`
}
```

Pronto, agora eu já vou conseguir usar essa struct no nosso serviço sem precisar duplicar código.

O próximo passo agora é criar o nosso serviço. Dentro de domínio, vou criar a pasta `person` e dentro dela o arquivo `person.go`

```go
// Arquivo: api-crud-salvando-arquivo/domain/person/person.go
package person

type Service struct {
    dbFilePath string
    people     domain.People
}
```

A primeira coisa que vou fazer é criar uma struct que armazenará as informações utilizadas pelo serviço, essas informações serão:

1 - o caminho do arquivo onde serão armazenadas as pessoas.

2 - as pessoas que estão armazenadas nesse arquivo.

No arquivo de entidades vou criar um tipo chamado `People` que será a representação do nosso arquivo de persistência.

```go
type People struct {
    People []Person `json:"people"`
}
```

Voltando pra criação do serviço responsável pela lógica de negócio de pessoas, eu vou receber o caminho completo do arquivo como parâmetro. Se o arquivo não existir, então eu crio um arquivo vazio. Se ele existir então eu vou pegar todas as pessoas armazenadas nele e colocar na variável people. Essa abordagem tá longe de ser a ideal porque vamos gastar bastante memória se o arquivo for grande, mas como no nosso caso aqui o objetivo é mostrar como manipular arquivos e construir uma API, eu não estou preocupado com essa otimização. Até porque, num caso real a gente provavelmente utilizaria um banco de dados ao invés de um arquivo como forma de persistência.

O primeiro passo é verificar se o arquivo passado já existe. O método `Stat` do pacote `os` nos dá informações sobre o arquivo e caso o arquivo não exista é retornado um erro. Esse erro pode ser usado no método IsNotExist para ver se o arquivo não existe. Se for esse o caso a gente cria o arquivo. Se o erro não for desse tipo então aconteceu algo inesperado que não sabemos como tratar, nesse caso vamos apenas retornar esse erro na criação do serviço.

```go
package person

type Service struct {
    dbFilePath string
    people     domain.People
}

func NewService(dbFilePath string) (Service, error) {
    _, err := os.Stat(dbFilePath)
    if err != nil {
        if os.IsNotExist(err) {
            err = createEmptyFile(dbFilePath)
            if err != nil {
                return Service{}, err
            }
            return Service{
                dbFilePath: dbFilePath,
                people:     domain.People{},
            }, nil
        } else {
            return Service{}, err
        }
    }

    jsonFile, err := os.Open(dbFilePath)
    if err != nil {
        return Service{}, fmt.Errorf("Error trying to open file that contains all people: %s", err.Error())
    }

    jsonFileContentByte, err := ioutil.ReadAll(jsonFile)
    if err != nil {
        return Service{}, fmt.Errorf("Error trying to read people file: %s", err.Error())
    }

    var allPeople domain.People
    json.Unmarshal(jsonFileContentByte, &allPeople)

    return Service{
        dbFilePath: dbFilePath,
        people:     allPeople,
    }, nil
}
```

A parte de criação de um arquivo vazio eu vou mover para um método separado, nesse método o primeiro passo
será criar uma variável do tipo People que vai ter uma slice vazia de pessoas, já que o arquivo não terá nenhuma pessoa inicialmente. Eu vou codificar essa variável para json utilizando o método `Marshal` do pacote `json`. Se der algum erro na codificação um erro será retornado.
Se der tudo certo eu vou salvar esse JSON no arquivo, pra isso vou utilizar a função `WriteFile` do pacote `ioutil`. Aqui eu vou passar as permissões do arquivo como 0755, mas na prática acho que poderia ter um pouco menos de permissão nesse caso. Se der erro eu vou retornar erro e se der tudo certo eu retorno `nil`.

```go
func createEmptyFile(dbFilePath string) error {
    var people domain.People = domain.People{
        People: []domain.Person{},
    }
    peopleJSON, err := json.Marshal(people)
    if err != nil {
        return fmt.Errorf("Error trying to encode people as JSON: %s", err.Error())
    }

    err = ioutil.WriteFile(dbFilePath, peopleJSON, 0755)
    if err != nil {
        return fmt.Errorf("Error trying to writing people file: %s", err.Error())
    }

    return nil
}
```

O que tá faltando agora é abrir o arquivo caso ele exista e colocar todas as pessoas que tem nesse arquivo dentro da variável people da nossa struct.
Primeiro é preciso abrir o arquivo com a função `Open` do pacote `os`.
Em seguida ler o conteúdo desse arquivo com o método `ReadAll` do pacote `ioutil` e jogá-lo pra uma nova variável.
E por fim decodificar esse JSON para uma variável da struct people com o `json.Unmarshal`.
Agora só retornar o serviço criado.

```go
    jsonFile, err := os.Open(dbFilePath)
    if err != nil {
        return Service{}, fmt.Errorf("Error trying to open file that contains all people: %s", err.Error())
    }

    jsonFileContentByte, err := ioutil.ReadAll(jsonFile)
    if err != nil {
        return Service{}, fmt.Errorf("Error trying to read people file: %s", err.Error())
    }

    var allPeople domain.People
    json.Unmarshal(jsonFileContentByte, &allPeople)

    return Service{
        dbFilePath: dbFilePath,
        people:     allPeople,
    }, nil
```

Pronto, agora o construtor do serviço já está implementado e podemos partir pra implementação do método de criação de pessoa.

```go
func (s *Service) Create(person domain.Person) error {
    // verifica se ja a pessoa ja existe, se já existe retorna erro

    s.people.People = append(s.people.People, person)
    // salvo o arquivo

    return nil
}
```

Para verificar se a pessoa já existe eu vou criar uma função que vai receber uma pessoa e com base no ID dela eu procuro na nossa slice de pessoas.

```go
func (s Service) exists(person domain.Person) bool {
    for _, currentPerson := range s.people.People {
        if currentPerson.ID == person.ID {
            return true
        }
    }
    return false
}
```

Pronto, agora eu já posso substituir na função de criação de pessoa a parte que tá com comentário fazendo essa verificação.

```go
func (s *Service) Create(person domain.Person) error {
    if s.exists(person) {
        return fmt.Errorf("There is already a person with this ID registered")
    }

    s.people.People = append(s.people.People, person)

    // salvar arquivo

    return nil
}
```

Por fim, basta criar uma função pra salvar o arquivo e chamar ela no lugar do comentário. Essa parte de salvar eu vou extrair pra uma função porque vou precisar na hora de implementar os outros métodos do CRUD.

```go
func (s *Service) Create(person domain.Person) error {
    if s.exists(person) {
        return fmt.Errorf("There is already a person with this ID registered")
    }

    s.people.People = append(s.people.People, person)
    err := s.saveFile()
    if err != nil {
        return fmt.Errorf("Error trying to add Person to file: %s", err.Error())
    }

    return nil
}

func (s Service) saveFile() error {
    allPeopleJSON, err := json.Marshal(s.people)
    if err != nil {
        return fmt.Errorf("Error trying to encode people as JSON: %s", err.Error())
    }
    return ioutil.WriteFile(s.dbFilePath, allPeopleJSON, 0755)
}
```

Pronto. A parte do serviço agora tá pronta.
Agora falta apenas criar esse serviço na função `main` e utilizar na nossa função que é chamada na rota de criação de pessoa.

```go
package main

import (
	"encoding/json"
	"fmt"
	"net/http"

	"github.com/brenoassp/api-crud-salvando-arquivo/domain"
	"github.com/brenoassp/api-crud-salvando-arquivo/domain/person"
)

func main() {
	personService, err := person.NewService("person.json")
	if err != nil {
		fmt.Printf("Error trying to creating personService: %s\n", err.Error())
	}

	http.HandleFunc("/person/", func(w http.ResponseWriter, r *http.Request) {
		if r.Method == "POST" {
			var person domain.Person
			err := json.NewDecoder(r.Body).Decode(&person)
			if err != nil {
				fmt.Printf("Error trying to decode body. Body should be a json. Error: %s\n", err.Error())
				http.Error(w, "Error trying to create person", http.StatusBadRequest)
				return
			}
			if person.ID <= 0 {
				http.Error(w, "person ID should be a positive integer", http.StatusBadRequest)
				return
			}

			err = personService.Create(person)
			if err != nil {
				fmt.Printf("Error trying to create person: %s\n", err.Error())
				http.Error(w, "Error trying to create person", http.StatusInternalServerError)
				return
			}
			w.WriteHeader(http.StatusCreated)
			return
		}
	})

	http.ListenAndServe(":8080", nil)
}
```

### Rota de atualização de pessoas (PUT /person/)

A rota de atualização de pessoas é, de certa forma, similar à rota de criação de pessoas.
Ela também receberá como payload as mesmas informações da pessoa, com a única diferença de que
ao invés de criar a pessoa, ela utilizará o ID fornecido para buscar a pessoa que precisa ser atualizada
no nosso arquivo. Caso não exista nenhuma pessoa com o ID fornecido, é preciso retornar um erro.

O primeiro passo é criar o código para tratar o método PUT no arquivo `main.go` que ficará quase igual
ao código de criação de pessoa, como pode ser visto abaixo:

```go
if r.Method == "PUT" {
    var person domain.Person
    err := json.NewDecoder(r.Body).Decode(&person)
    if err != nil {
        fmt.Printf("Error trying to decode body. Body should be a json. Error: %s\n", err.Error())
        http.Error(w, "Error trying to update person", http.StatusBadRequest)
        return
    }
    if person.ID <= 0 {
        http.Error(w, "person ID should be a positive integer", http.StatusBadRequest)
        return
    }

    err = personService.Update(person)
    if err != nil {
        fmt.Printf("Error trying to update person: %s\n", err.Error())
        http.Error(w, "Error trying to update person", http.StatusInternalServerError)
        return
    }
    w.WriteHeader(http.StatusOK)
    return
}
```

Em seguida, é preciso criar o método responsável pela atualizar da pessoa no serviço. Antes de fazer a atualização
é preciso verificar se existe uma pessoa com o ID fornecido para fazer a atualização, caso contrário um erro é gerado.
Se o registro foi encontrado, basta atualizar a slice de pessoas e salvar o arquivo.
Um ponto importante aqui é a necessidade de utilizar ponteiro na assinatura do método pois estamos alterando a slice,
caso contrário iríamos estar alterando a cópia e as futuras chamadas de métodos do serviço utilizaria dados desatualizados com a
realidade do nosso arquivo.

```go
func (s *Service) Update(person domain.Person) error {
	var indexToUpdate int = -1
	for index, currentPerson := range s.people.People {
		if currentPerson.ID == person.ID {
			indexToUpdate = index
			break
		}
	}
	if indexToUpdate < 0 {
		return fmt.Errorf("There is no person with the given ID to be updated")
	}

	s.people.People[indexToUpdate] = person
	return s.saveFile()
}
```

### Rota de listagem de pessoas (GET /person/ e GET /person/{id})

O primeiro passo a fazer na rota de listagem de pessoas é a distinção entre
a rota de listagem de uma única pessoa e a rota de listagem de todas as pessoas
do nosso arquivo. Para isso, vamos utilizar o método `TrimPrefix` passando o
que veio no PATH da URL e o prefixo que queremos excluir `/person/`, com isso
teremos dois possíveis resultados:

1 - a string vazia `""` se não tiver nada após o prefixo.

2 - uma string com o conteúdo contido após a string `/person/`.

No primeiro caso significa que precisamos retornar todas as pessoas.

Já no segundo precisamos verificar se o conteúdo é um ID válido e, caso for, buscar a pessoa
que possui esse ID no nosso arquivo. Primeiro tentamos converter para inteiro essa string com
o método `Atoi` do pacote `strconv`, se der errado então já sabemos que não é um ID válido e
podemos responder com um erro. Caso a conversão aconteça com sucesso, fazemos a validação se
o ID fornecido é um número inteiro positivo. Por fim, se passar em todas as validações,
estamos aptos a buscar a pessoa com o ID fornecido.

```go
if r.Method == "GET" {
    path := strings.TrimPrefix(r.URL.Path, "/person/")
    if path == "" {
        // list all people
        w.WriteHeader(http.StatusOK)
        w.Header().Set("Content-Type", "application/json")
        err = json.NewEncoder(w).Encode(personService.List())
        if err != nil {
            http.Error(w, "Error trying to list people", http.StatusInternalServerError)
            return
        }
    } else {
        personID, err := strconv.Atoi(path)
        if err != nil {
            http.Error(w, "Invalid id given. person ID must be an integer", http.StatusBadRequest)
            return
        }
        person, err := personService.GetByID(personID)
        if err != nil {
            http.Error(w, err.Error(), http.StatusNotFound)
            return
        }
        w.WriteHeader(http.StatusOK)
        w.Header().Set("Content-Type", "application/json")
        err = json.NewEncoder(w).Encode(person)
        if err != nil {
            http.Error(w, "Error trying to get person", http.StatusInternalServerError)
            return
        }
    }
    return
}
```

No nosso serviço nós teremos duas funções diferentes, uma para cada um dos cenários acima.
No caso de não ser passado nenhum ID, iremos listar todas as pessoas do nosso arquivo. Como
o código que fizemos foi feito de tal forma a deixar sempre a slice do serviço atualizada
com o nosso arquivo, basta retornar o conteúdo dessa slice quando quisermos saber quais
são as pessoas presentes no arquivo.

Já no caso de retornar uma pessoa com base no ID, é preciso buscar na slice se existe
alguma pessoa com o ID fornecido e retorná-la caso a encontre. Se não for o caso, basta
retornar um erro de `Pessoa não encontrada`.

```go
func (s Service) List() domain.People {
	return s.people
}

func (s Service) GetByID(personID int) (domain.Person, error) {
	for _, currentPerson := range s.people.People {
		if currentPerson.ID == personID {
			return currentPerson, nil
		}
	}
	return domain.Person{}, fmt.Errorf("Person not found")
}
```

### Rota de deleção de pessoas (DELETE /person/{id})

Para a rota de deleção de pessoas iremos exigir que um ID seja passado na URL para
identificar qual a pessoa que precisará ser deletada. Com base nesse ID, iremos
procurar a pessoa no nosso arquivo para deletá-la caso exista. A validação do ID
é feita da mesma forma que foi feita na rota de listagem de pessoas.

```go
if r.Method == "DELETE" {
    path := strings.TrimPrefix(r.URL.Path, "/person/")
    if path == "" {
        http.Error(w, "ID is required to delete a person", http.StatusBadRequest)
        return
    } else {
        personID, err := strconv.Atoi(path)
        if err != nil {
            http.Error(w, "Invalid id given. person ID must be an integer", http.StatusBadRequest)
            return
        }
        err = personService.DeleteByID(personID)
        if err != nil {
            fmt.Printf("Error trying to delete person: %s\n", err.Error())
            http.Error(w, "Error trying to delete person", http.StatusInternalServerError)
            return
        }
        w.WriteHeader(http.StatusOK)
    }
    return
}
```

Será necessário criar o método `DeleteByID` no serviço para deletar a pessoa com o ID passado.
Assim como no método de `Update`, é necessário utilizar o ponteiro na assinatura do método para
realizar a alteração da slice original.

O primeiro passo é pesquisar na slice se existe uma pessoa com o ID fornecido, caso não exista,
um erro de pessoa não encontrada deve ser retornado. Caso a pessoa seja encontrada, criamos uma
nova slice que não contenha a pessoa que deve ser deletada e a armazenamos na variável people do
nosso serviço. Por fim, atualizamos nosso arquivo com o conteúdo novo dessa variável com o método
`saveFile()`.

```go
func (s *Service) DeleteByID(personID int) error {
	var indexToRemove int = -1
	for index, currentPerson := range s.people.People {
		if currentPerson.ID == personID {
			indexToRemove = index
			break
		}
	}
	if indexToRemove < 0 {
		return fmt.Errorf("There is no person with the provided ID")
	}

	s.people.People = append(
		s.people.People[:indexToRemove],
		s.people.People[indexToRemove+1:]...,
	)

	return s.saveFile()
}
```
