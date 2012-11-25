---
published: true
layout: post
title: Especificações de Views com MiniTest
date: 2012-11-25 19:30
comments: true
categories:
 - Rails
 - TDD
 - MiniTest
 - Test Driven Development
---

As Views representam a Interface do Usuário de uma aplicação. Como tal, ficam sujeitas a frequentes mudanças, orientadas por
preocupações de usabilidade, de estética ou mesmo de evolução de requisitos.

No Rails, as views normalmente são arquivos html contendo código Ruby focado em apresentar dados para o usuário.
Via de regra, estes dados provêm de Models que foram recuperados por um Controller. Quando desenvolvemos _de fora para dentro_
(_outside-in development_), as necessidades das views orientam o projeto dos controllers e models, levando a APIs mais
consistentes e bem alinhadas com o comportamento desejado para a aplicação.

Tamanha a responsabilidade das Views, que elas não deveriam ser ignoradas nas atividades de teste. Este artigo ilustra
uma abordagem para testar views e alguns pontos que o desenvolvedor deve estar atento quando escreve este tipo de teste.

<!-- more -->

View Specs
----------

A MiniTest ainda não oferece oficialmente suporte a specs de views. Isto fica evidente por dois motivos:

 * ao fazer scaffold, seu gerador de código não cria um diretório ```test/views```
 * não há exemplos de view specs que possam servir de referênca para o desenvolvedor

Existe, porém, um esforço nesse sentido, como pode ficar claro
[nesta thread](https://groups.google.com/forum/?fromgroups=#!topic/minitest-rails/pJQNRZeua3U%5B1-25%5D).
Estudando um pouco o código-fonte do minitest-rails, é possível encontrar
[pistas](https://github.com/blowmage/minitest-rails/blob/91297faee6b008473aac0c39192d196331fd3caa/lib/minitest/rails/action_view.rb)
de como possibilitar a escrita de testes de views: se o nome do teste atender à expressão regular ```/(Helper|View)( ?Test)?\z/i```
este teste será executado como um ```MiniTest::Rails::ActionView::TestCase```.

Partindo do post anterior no qual foi criada uma aplicação de Quiz e foi feito scaffold de ```Question```, crie o arquivo:

``` ruby test/views/questions_view_test.rb
require 'minitest_helper'

describe 'Questions View Test' do
  it "is a view spec"
end
```

Note que:

 * a descrição do teste ```Question View Test``` obedece à expressão regular de view specs,
 * foi fornecido um teste ```it "is a view spec"``` mas ainda não foi escrito o código desse teste.

Veja a saída de sua execução:

``` bash Terminal
$ rake minitest:views

Run options: --seed 63869
# Running tests:
S
Finished tests in 0.201182s, 4.9706 tests/s, 0.0000 assertions/s.
1 tests, 0 assertions, 0 failures, 0 errors, 1 skips
```

O ```S``` significa que o teste foi ignorado (s de _Skipped_).
Quando um teste é descrito sem uma implementação, a sinalização ```S``` indica que ele está pendente. Este recurso é
muito útil, pois serve para a criação de _TODO Lists_ enquanto são modeladas as especificações de teste da aplicação.

Para que as view specs sejam executadas junto com os demais testes pelo comando ```rake minitest:all```, é necessário
alterar a [variável global MINITEST_TASKS](https://github.com/blowmage/minitest-rails/blob/91297faee6b008473aac0c39192d196331fd3caa/lib/minitest/rails/tasks/minitest.rake),
no ```Rakefile```:

``` ruby Rakefile
#!/usr/bin/env rake
# Add your own tasks in files placed in lib/tasks ending in .rake,
# for example lib/tasks/capistrano.rake, and they will automatically be available to Rake.

require File.expand_path('../config/application', __FILE__)

Quizz::Application.load_tasks

MINITEST_TASKS << "views"
```

Configurado o MiniTest para rodar view specs, boa hora para comitar no git:

``` bash Terminal
git add .
git commit -m "Configura MiniTest para rodar view specs"
```

Afinal, o que testar nas Views?
-------------------------------

É muito importante, em todo tipo de teste, que se tenha um foco claro daquilo que precisa ser testado. Tome-se como exemplo
uma página de listagem de questões. Pode-se listar facilmente alguns comportamentos para ela:

  * Quando não houver questão cadastrada
    * Deve exibir uma mensagem "Nenhuma questão cadastrada"
    * Não deve exibir a tabela de questões
  * Quando houver questões
    * Não deve exibir a mensagem "Nenhuma questão cadastrada"
    * Deve exibir a tabela de questões

Note que as verificações listadas acima __têm por objetivo testar se a view representa corretamente o estado da aplicação__.
Ou seja, fornecendo-se para a view os dados num determinado estado (__sem cadastros__ ou __com cadastros__), deve-se verificar se
ela exibe/omite corretamente uma mensagem ou uma tabela.

Os testes podem ser inicialmente escritos na forma de uma _TODO List_: desta forma tem-se sempre um roteiro de trabalho bem
definido e fica mais fácil manter o foco na tarefa atual, sempre em [_passinhos de bebê_](http://improveit.com.br/xp/principios/passos_bebe).

``` ruby test/views/questions_view_test.rb
require 'minitest_helper'

describe 'Questions View Test' do
  describe "index" do
    describe "with no questions" do
      it "shows empty message"
      it "omits table"
    end

    describe "with saved questions" do
      it "omits empty message"
      it "shows table"
    end
  end
end
```
Antes de prosseguir com a implementação de cada teste, um commitzinho cai bem:

``` bash Terminal
git add .
git commit -m "Adiciona spec inicial de Questions View"
```

E agora, José? Como se testa isso?
----------------------------------

Nas views, quase sempre é preciso atentar para:

 * Fornecer dados para a view no estado que se deseja verificar;
 * Renderizar a view e verificar se ela apresentou corretamente o estado que ela recebeu;
   * Selecionar elementos de marcação (_markup_) que têm valor semântico para a aplicação.

No artigo anterior, via scaffold, foram geradas as views, dentre as quais o foco do teste atual é:

``` html app/views/questions/index.html.erb
<h1>Listing questions</h1>
<table>
  <tr>
    <th>Title</th>
    <th>Content</th>
    <th>Difficulty</th>
    <th></th>
    <th></th>
    <th></th>
  </tr>
<% @questions.each do |question| %>
  <tr>
    <td><%= question.title %></td>
    <td><%= question.content %></td>
    <td><%= question.difficulty %></td>
    <td><%= link_to 'Show', question %></td>
    <td><%= link_to 'Edit', edit_question_path(question) %></td>
    <td><%= link_to 'Destroy', question, method: :delete, data: { confirm: 'Are you sure?' } %></td>
  </tr>
<% end %>
</table>
```

Repare bem na linha 11: a view renderiza os dados a partir da variável de instância ```@questions```
fornecida para ela. É através desta variável, portanto, que será configurado o estado do teste. Veja como
fica o setup (preparação) do teste:

``` ruby Setup da spec "with no questions"
...
describe "with no questions" do
  before do
    @questions = []
    render file: "questions/index"
    puts rendered # apenas para fins de demonstração
  end
  ...
end
```

Ao fazer ```@questions``` ser um array vazio, coloca-se a variável no estado que se deseja testar
(__nenhuma questão cadastrada__). Na sequência, o método ```render``` renderiza a view ```questions/index```.
O conteúdo da view pode ser então acessado via chamada ```rendered```.

Por fim, o teste em si, para verificar se o conteúdo renderizado está de acordo com o comportamento esperado:

``` ruby Verificações de conteúdo com assert_select
it "shows empty message" do
  assert_select "p.info", "Nenhuma questao cadastrada"
end

it "omits table" do
  assert_select "table", 0
end
```
O ```assert_select``` [pode ser usado de várias formas](http://apidock.com/rails/ActionDispatch/Assertions/SelectorAssertions/assert_select)
para realizar verificações no conteúdo de ```rendered```.

No primeiro teste, ele verifica se a view renderizou um parágrafo com a classe CSS ```.info```, e se contém a
respectiva mensagem.

No segundo, verifica se algum elemento ```table``` foi renderizado. Caso afirmativo, o teste falha, pois
o ```assert_select``` recebeu como segundo argumento um número, representando quantas vezes é esperado que
o elemento ```table``` ocorra - neste caso, zero!

Veja a saída dos testes:

``` bash Terminal
$ rake minitest:views
Run options: --seed 50168
# Running tests:
FFSS
Finished tests in 0.143716s, 27.8327 tests/s, 13.9163 assertions/s.

  1) Failure:
test_0002_omits table(Questions View Test::index::with no questions) [/Users/fabio/workspace/miranti/quizz/test/views/questions_view_test.rb:16]:
Expected exactly 0 elements matching "table", found 1.

  2) Failure:
test_0001_shows empty message(Questions View Test::index::with no questions) [/Users/fabio/workspace/miranti/quizz/test/views/questions_view_test.rb:12]:
Expected at least 1 element matching "p.info", found 0.
```

Excelente! Com os testes falhando, estamos no <span style="color: red">Vermelho</span> do ciclo
<span style="color: red">Vermelho</span> / <span style="color: limegreen">Verde</span> / <span style="color: brown">Refatore</span>
[_Red/Green/Refactor_](http://rtiweb.net/engenheiro-software/2011/05/18/tutorial-tdd-%E2%80%93-test-drive-development/#axzz2DGv3nW87).
É hora de fornecer <strong><em>a implementação <span style="font-size: larger">mais simples</span></em></strong> que faça os testes passarem:
``` ruby app/views/questions.index.html.erb
<p class="info">Nenhuma questao cadastrada</p>
<% unless @questions.empty? %>
<table>
  ...
</table>
<% end %>
```

Os testes passam:
``` bash Terminal
$ rake minitest:views
Run options: --seed 54895
# Running tests:
..SS
Finished tests in 0.301827s, 13.2526 tests/s, 6.6263 assertions/s.
4 tests, 2 assertions, 0 failures, 0 errors, 2 skips
```

Estamos no <span style="color: limegreen">verde</span>, e fazendo progressos! Com gestos de
[cavalaria](http://www.youtube.com/watch?v=GyT_KyAqDEc#t=70), cantamos: Oppa Commit Time!

``` bash Terminal
git commit -am "Tratamento da view questions/index quando não há questões cadastradas"
```


Repetindo o processo, para o cenário seguinte, o teste fica escrito por:

``` ruby Spec "com questões cadastradas"
describe "with five questions" do
  before do
    @questions = (1..5).each.collect {|i|
      question = Question.new title: "Title #{i}",
        content: "Mussum Impsum #{i}", difficulty: i

      # estamos simulando uma questão salva no banco de dados, logo, precisa ter um id
      question.id = i
      question
    }
    render file: "questions/index"
  end

  it "omits empty message" do
    assert_select "p.info", count: 0,
      text: "Nenhuma questao cadastrada"
  end

  it "shows table" do
    assert_select "table" do
      assert_select "thead tr", 1
      assert_select "tbody tr", @questions.count
    end
  end
end
```
Repare nas verificações:

 * Para a ausência da mensagem "Nenhuma questão cadastrada": o ```assert_select``` recebe um hash,
 com o atributo ```count``` informando que são esperados "zero elementos" ```p.info``` com o texto citado.
 * Para os elementos da tabela: o ```assert_select``` permite __<em>aninhamento</em> de verificações__, ou seja, uma verificação
 dentro de outra verificação. Primeiro é feita a seleção da tabela - caso ela seja encontrada, aí sim
 verifica-se a presença de linhas da tabela, ```thead tr``` e ```tbody tr```.

Os testes falham, conforme era de se esperar:
``` bash Terminal
$ rake minitest:views
Run options: --seed 57692
# Running tests:
FF..
Finished tests in 0.367636s, 10.8803 tests/s, 13.6004 assertions/s.

  1) Failure:
test_0002_shows table(Questions View Test::index::with five questions) [/Users/fabio/workspace/miranti/quizz/test/views/questions_view_test.rb:37]:
Expected exactly 1 element matching "thead tr", found 0.

  2) Failure:
test_0001_omits empty message(Questions View Test::index::with five questions) [/Users/fabio/workspace/miranti/quizz/test/views/questions_view_test.rb:32]:
Expected exactly 0 elements matching "p.info", found 1.
```

Segue a implementação que faz os testes passar:
``` ruby Versão final de app/views/questions/index.html.erb
<% if @questions.empty? %>
  <p class="info">Nenhuma questao cadastrada</p>
<% else %>
<table>
  <thead>
    <tr>
      <th>Title</th>
      <th>Content</th>
      <th>Difficulty</th>
      <th></th>
      <th></th>
      <th></th>
    </tr>
  </thead>
<% @questions.each do |question| %>
  <tbody>
    <tr>
      <td><%= question.title %></td>
      <td><%= question.content %></td>
      <td><%= question.difficulty %></td>
      <td><%= link_to 'Show', question %></td>
      <td><%= link_to 'Edit', edit_question_path(question) %></td>
      <td><%= link_to 'Destroy', question, method: :delete, data: { confirm: 'Are you sure?' } %></td>
    </tr>
  </tbody>
<% end %>
</table>
<% end %>
```

E passaram mesmo, ó:
``` bash Terminal
$ rake minitest:views
Run options: --seed 24562
# Running tests:
....
Finished tests in 0.118319s, 33.8069 tests/s, 50.7104 assertions/s.
4 tests, 6 assertions, 0 failures, 0 errors, 0 skips
```

Por fim, um commitão, só pra variar, mas agora seguido de um push pro GitHub.
``` bash Terminal
git commit -am "Tratamento da view questions/index quando há questões cadastradas"
git push origin master
```

Considerações Finais
--------------------
#### Muito teste pra pouco código?
Neste artigo foram geradas 41 linhas de código de teste, como pode ser visto
[neste commit](https://github.com/fabiolnm/quiz/blob/16ea3bf9f236bbd0c4e57b4d4057e19b018e4a6a/test/views/questions_view_test.rb). No entanto,
foram adicionadas apenas [7 linhas de código na view](https://github.com/fabiolnm/quiz/commit/16ea3bf9f236bbd0c4e57b4d4057e19b018e4a6a).
Uma proporção de 1:6. O custo do código de testes realmente compensa o investimento em escrevê-los?

Tradicionalmente, as aplicações são testadas levantando-se um servidor e fazendo inspeção _manual_ do funcionamento da aplicação, por meio de
navegação em suas páginas. Muitas aplicações carecem também de longas etapas de homologação, onde cada funcionalidade precisa ser validada
manualmente a cada release.

O teste automatizado não elimina os testes manuais - vide o artigo [Testing is Overrated](http://railspikes.com/2008/7/11/testing-is-overrated) -
mas ajuda a manter o foco nas funcionalidades relevantes, e tornando a homologação mais ágil, com menos riscos.

Outra forma de pensar é: o Rails fornece muitos recursos que contribuem para o ganho de agilidade e produtividade (por exemplo, desde o scaffold,
já está disponível uma aplicação funcional, criada com uma única linha de comando). É bastante prudente utilizar com sabedoria essas facilidades,
e aproveitar o tempo que se ganhou como um bônus para investir em qualidade e confiabilidade dos softwares construídos com esta fantástica framework!

#### Em nenhum momento foi aberto o navegador
Desde que a aplicação Quiz foi criada no post anterior, em nenhum momento foi necessário abrir o navegador. Isso é interessantíssimo:
consegue-se evoluir as funcionalidades de uma aplicação Web sem a realização de testes manuais, sem a necessidade de navegação humana
para assegurar que a aplicação está funcionando.

Ao automatizar os testes de views, deve-se procurar um ponto de equilíbrio, onde os testes repetitivos possam ser automatizados, e os testes
manuais possam ser direcionados e focados nos aspectos mais relevantes/críticos da aplicação.

#### Proporção 1:3
É muito comum em aplicações Rails bem escritas que a relação entre a quantidade de código da aplicação e a quantidade de código de teste da aplicação
fique em torno de [1:2 e 1:3](http://37signals.com/svn/posts/3159-testing-like-the-tsa).

Se você desenvolvedor ainda não aderiu à disciplina de testes, é melhor começar a se sentir
<strong style="font-size: larger">CULPADO</strong>, pois está fazendo apenas <strong style="font-size: larger">UM TERÇO</strong> de seu trabalho.

Se você contratante ainda recruta desenvolvedores que não praticam a disciplina de testes, deveria ficar <strong style="font-size: larger">PREOCUPADO</strong>,
pois está pagando <strong style="font-size: larger">TRÊS VEZES MAIS CARO</strong>, e ainda corre um alto risco de receber um trabalho com baixa qualidade.

#### Teste focado e desacoplado
A [Arquitetura MVC](http://guides.rubyonrails.org/getting_started.html#the-mvc-architecture) encoraja uma clara organização de código.
Os testes de views mostrados neste artigo estão totalmente alinhados com este princípio. Sendo o papel das views apresentar para o usuário
o estado da aplicação, a preparação (setup) dos testes coloca a aplicação no estado que se deseja testar, sem interação nem com os
controllers nem com o banco de dados. O teste é, portanto, focado em verificar apenas se a View está cumprindo seu papel.

Este desacoplamento é importantíssimo para a saúde da suíte de testes. Quando o acoplamento é alto, à medida que a aplicação cresce em
complexidade, é muito comum que uma pequena alteração de código provoque a quebra de muitos testes da suíte. Isto torna caras
até mesmo pequenas alterações, impactando na agilidade e na produtividade, alguns dos benefícios esperados quando se adota TDD.

No [capítulo 6 do Livro do Cucumber, versão P1.0](http://pragprog.com/book/hwcuc/the-cucumber-book), este sintoma é catalogado como
<string><em>Funcionalidade Frágil (Brittle Feature)</em></strong>. É uma dívida técnica que tende a aumentar rapidamente, pois quando isso
ocorre a tendência é tirar os testes do caminho, ou ter menos cuidado na escrita de novos testes.


#### Nenhum acesso ao banco de dados
Em todas as preparações (setups) de teste de views, os objetos de modelos foram instanciados em memória, simulando o estado que eles teriam na
aplicação. Para simular um objeto "salvo no banco de dados", bastou setar o id do objeto, como se ele tivesse vindo do banco.

Isto torna os testes de views independentes dos arquivos de [_fixtures_](http://guides.rubyonrails.org/testing.html#the-low-down-on-fixtures)
de teste da aplicação. Normalmente esses testes também acabam rodando mais rápido, dado que não é feito nenhum acesso a banco. Isso contribui
para a rapidez e a saúde da suíte de testes.

#### Modelagem Semântica da Página
Outro benefício de fazer testes de views é que cada página acaba sendo construída de acordo com as preocupações
semânticas direcionadas pelos testes. Uma mensagem "Nenhum resultado encontrado" parece ficar bem em um parágrafo
com classe CSS info. Dados provenientes de bancos de dados normalmente se adaptam bem a estruturas tabulares. Mensagens de erro
normalmente são fortes candidatas a renderizar em listas, e por aí vai.

Este cuidado com organização semântica é importante inclusive no que tange às atividades de SEO. Além disso, é muito
mais fácil e menos caro um desenvolvedor entregar para um Web Designer um markup pronto, enxuto, para que o Web Designer
possa focar na criação das folhas de estilo que deixam o sistema mais elegante, do que o contrário - o desenvolvedor
pegar um design pronto e transformá-lo em algo funcional e que tenha significado para a aplicação.

O [CSS Zen Garden](http://www.mezzoblue.com/zengarden/alldesigns/) é um exemplo que vale à pena citar de como é importante
um sistema ter um markup enxuto e consistente, que os web designers têm a liberdade de estilizar, sem precisar alterar
o markup.

Bibliografia
------------
 * [The MVC Architecture](http://guides.rubyonrails.org/getting_started.html#the-mvc-architecture)
 * [The RSpec Book, version P2.1, chapter 23](http://pragprog.com/book/achbd/the-rspec-book)
 * [API Dock - assert_select](http://apidock.com/rails/ActionDispatch/Assertions/SelectorAssertions/assert_select)
 * [ActionView::TestCase - a base para testes de views](https://github.com/rails/rails/blob/e95b9d6c68b1e0bba3840d18fc0aa94ccf88776d/actionpack/lib/action_view/test_case.rb)
 * [ActionView::TestCase::Behavior - a ponte entre as variáveis de instância do teste e da view,
 e que fornece o método rendered](http://api.rubyonrails.org/classes/ActionView/TestCase/Behavior.html#method-i-locals)
 * [Documentação do método render, usado nos testes de views](http://api.rubyonrails.org/classes/ActionView/Helpers/RenderingHelper.html#method-i-render)
 * [CSS Zen Garden](http://www.mezzoblue.com/zengarden/alldesigns/)
 * [Testing is Overrated](http://railspikes.com/2008/7/11/testing-is-overrated)
 * [Testing like the TSA](http://37signals.com/svn/posts/3159-testing-like-the-tsa)
