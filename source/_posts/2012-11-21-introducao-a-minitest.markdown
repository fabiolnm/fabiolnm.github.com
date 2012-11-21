---
published: true
layout: post
title: Introdução à MiniTest
date: 2012-11-21 20:00
comments: true
categories:
 - Rails
 - TDD
 - MiniTest
 - Test Driven Development
---

O framework Ruby-on-Rails possui muitas facilidades para a escrita de testes de aplicações web.
Historicamente, a API Test::Unit provê os recursos básicos, e muitas outras gems surgiram
para complementar ou mesmo substituir o Test::Unit (vide RSpec, Shoulda, Mocha, etc).

Em 2010, a versão 1.9 do Ruby introduziu uma versão atualizada da Test::Unit, chamada MiniTest.
Ao mesmo tempo que guarda retrocompatibilidade com o Test::Unit, a MiniTest introduz novos recursos
à linguagem, sendo a principal delas uma DSL (Domain Specific Language) para escrever testes num
estilo BDD (Behavior Driven Development), similar à sintaxe do RSpec.

<!-- more -->

Isto constitui um passo importante dentro da biblioteca-base do Ruby, agora numa posição mais neutra
na [controvérsia TDD x BDD](http://akitaonrails.com/2011/04/17/a-controversia-test-unit-vs-rspec-cucumber).
No Rails 4, a [Test::Unit deverá subclassear MiniTest::Spec](https://github.com/rails/rails/blob/master/activesupport/lib/active_support/test_case.rb),
para se ter uma idéia de quão significativa é essa mudança.

Iniciando um Projeto, sem a Test::Unit
--------------------------------------

O comando ```rails new nome_do_projeto```, por padrão, cria um diretório com a estrutura padrão do Test::Unit:

    ▾ test/
      ▾ fixtures/
      ▾ functional/
      ▾ integration/
      ▾ performance/
      ▾ unit/
      test_helper.rb

Como pretendemos usar a MiniTest, podemos criar o projeto passando o argumento -T, equivalente a --skip-test-unit,
ou seja, cria o projeto, mas pulando a criação da estrutura básica do Test::Unit:

``` bash Terminal
rails new quiz -T
```

Criado o projeto, é um bom momento para versionar com o Git
``` bash Terminal
git init
git add .
git commit -m "Commit inicial"
```

Apesar deste post não ser sobre git, vamos seguir as boas práticas e dar passos pequenos e consistentes,
comitando cada avanço que seja significativo.

MiniTest-Rails
--------------
A gem [minitest-rails](https://github.com/blowmage/minitest-rails) vem suprindo as extensões necessárias para
usar a MiniTest em projetos Rails. É necessário adicioná-la ao Gemfile:

``` ruby Gemfile
group :development, :test do
  gem 'minitest-rails'
end
```

Em seguida, é necessário instalar a gem com o bundler, e utilizar um rails generator para
criar o arquivo test/minitest_helper.rb:
``` bash Terminal
bundle install
rails generate mini_test:install
```

Bom momento pra comitar:
``` bash Terminal
git add .
git commit -m "Instala minitest-rails"
```

Podemos então prosseguir para a adição da primeira funcionalidade de um Quiz: a criação de Perguntas.
Para acelerar o desenvolvimento, nada melhor que utilizar o rails scaffold, que cria toda uma estrutura MVC como
ponto de partida. Antes de rodar o scaffold, porém, devemos configurar o generator para gerar os testes utilizando
a DSL do MiniTest::Spec:

``` ruby config/application.rb
config.generators do |g|
  g.test_framework :mini_test, spec: true
end
```

Ao rodar o generator:

``` bash Terminal
rails generate scaffold Question title:string content:text difficulty:integer
```

É gerada a estrutura básica da aplicação. Atenção especial para o diretório de testes:

    ▾ test/
      ▾ controllers/
          questions_controller_test.rb
      ▾ fixtures/
          questions.yml
      ▾ helpers/
          questions_helper_test.rb
      ▾ models/
          question_test.rb

Foi gerada, portanto, a estrutura da aplicação, e os respectivos testes para os
Models, Controllers e Views que foram gerados. Antes de mergulhar a fundo em cada um desses testes,
vale a pena rodar a suite de testes. Primeiro, roda-se a migration de criação da tabela questions no
banco de dados:

``` bash terminal - execução da migration criada no scaffold
rake db:migrate
```

Uma vez que não haja mais nenhuma migration pendente, os testes podem ser executados com um dos comandos:
``` bash Terminal
rake test                 #
rake minitest             # rodam todos os testes
rake minitest:all         #

rake minitest:models      # roda apenas os testes de unidade dos models
rake minitest:helpers     # roda apenas os testes dos helpers das views
rake minitest:controllers # roda apenas os testes funcionais dos controllers
```

Veja abaixo saída da execução dos testes. Primeiro são executados os testes de models, depois os helpers e por fim os funcionais.
Cada ```.``` após a linha ```# Running tests``` é um teste executado com sucesso. Cada ```F```, um teste falhando. Os testes funcionais
gerados via scaffold são 7, um para cada [ação de um recurso RESTful](http://guides.rubyonrails.org/routing.html#crud-verbs-and-actions).

``` bash Terminal
$ rake minitest:all

Run options: --seed 27504
# Running tests:
.
Finished tests in 0.063599s, 15.7235 tests/s, 15.7235 assertions/s.
1 tests, 1 assertions, 0 failures, 0 errors, 0 skips

Run options: --seed 3609
# Running tests:
F
Finished tests in 0.070579s, 14.1685 tests/s, 14.1685 assertions/s.
  1) Failure:
test_0001_must be a real test(QuestionsHelper) [/Users/fabio/workspace/miranti/quizz/test/helpers/questions_helper_test.rb:6]:
Need real tests
1 tests, 1 assertions, 1 failures, 0 errors, 0 skips

Run options: --seed 4676
# Running tests:
.......
Finished tests in 0.150315s, 46.5689 tests/s, 66.5270 assertions/s.
7 tests, 10 assertions, 0 failures, 0 errors, 0 skips

Errors running minitest:helpers! #<RuntimeError: Command failed with status (1): [ruby -I"lib:test" -I"/Users/fabio/.rbenv/versions/1.9.3-p286/lib/ruby/gems/1.9.1/gems/rake-10.0.1/lib" "/Users/fabio/.rbenv/versions/1.9.3-p286/lib/ruby/gems/1.9.1/gems/rake-10.0.1/lib/rake/rake_test_loader.rb" "test/helpers/**/*_test.rb" ]>
```

Para finalizar esta parte introdutória, podemos fazer os commits no git:
``` bash Terminal
git add config/application.rb
git commit -m "Configura gerador de código do MiniTest"

git add .
git commit -m "Adiciona scaffold do modelo Question"
```

E fazer um push para um repositório remoto, por exemplo o [GitHub](https://github.com):
``` bash Terminal
git remote add origin git@github.com:fabiolnm/quiz.git
git push -u origin master
```

Conclusão
---------
Neste post introdutório:

  * Iniciamos uma aplicação sem a Test::Unit
  * Instalamos e configuramos a MiniTest
  * Geramos algumas specs via rails scaffolding
  * Rodamos os testes com o Rake

Nos próximos posts desta série, serão explorados em maior profundidade cada um dos tipos de spec - de models, views e controllers.

Bibliografia
------------
 * [A Guide to Testing Rails Applications](http://guides.rubyonrails.org/testing.html)
 * [MiniTest: a framework de teste do Ruby 1.9](http://www.bootspring.com/2010/09/22/minitest-rubys-test-framework)
 * [A Path to Rails 4 with MiniTest::Spec](http://www.rubyflow.com/items/8037-a-path-to-rails-4-with-minitest-spec)
 * [Using MiniTest::Spec with Rails](http://metaskills.net/2011/03/26/using-minitest-spec-with-rails)
 * [A controvérsia Test::Unit vs RSpec/Cucumber](http://akitaonrails.com/2011/04/17/a-controversia-test-unit-vs-rspec-cucumber)
 * [A better way of testing Rails application with minitest](http://blog.rawonrails.com/2012/01/better-way-of-testing-rails-application.html)
