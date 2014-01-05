---
published: true
layout: post
title: "Dependency Injection em Ruby"
date: 2014-01-05 02:45
comments: true
categories:
 - Ruby
 - Java
 - Design Patterns
---

O padrão de projeto Dependency Injection se popularizou bastante entre desenvolvedores Java,
por facilitar o teste de uma classe e seus colaboradores, ao remover dependências fortes
entre eles. Mas este padrão pode ser globalmente aplicado em outras linguagens? Ou seria
um indício de um defeito da linguagem?

<!-- more -->
Quando trabalhava com Java e tive a oportunidade de me aprofundar na prática de testes,
DI fornecia uma maneira elegante de manipular os colaboradores de uma classe, tornando
possível a verificação da maneira como os objetos interagiam entre si.

Um cenário real onde pude aplicar o pattern foi num teste de integração entre sistemas.
Havia um modelo que era responsável por persistir mídias (fotos / videos) num sistema A,
e notificar uma API de outro sistema B, para processar aquelas mídias.

Uma implementação sem DI:

``` java
class ProcessadorMidiaNoSistemaA {
  private NotificadorSistemaB notificador = new NotificadorSistemaB();  // (1)

  void processarMidia(Midia midia) {
    persisteMidiaLocalmente(midia);
    notificador.envia(midia);                                           // (2)
  }
}
```

Esta implementação possui dois inconvenientes do ponto de vista de testes:

1. não é possível, de maneira simples, acessar o colaborador que notifica o sistema B,
para realizar verificações;
2. ao exercitar o método processarMidia, um teste inevitavelmente invocaria o notificador
real. O impacto para a suíte de testes poderia se refletir em:

  2.1. testes mais lentos, caso a notificação seja demorada; ou

  2.2. testes frágeis, caso o sistema externo falhe (por exemplo fique fora do ar durante
  o teste). Neste caso, o sistema estaria funcionando corretamente, com os comportamentos
  esperados, mas a suíte quebraria por falhas que não são de sua responsabilidade.

A solução para este cenário, usando Dependency Injection:

``` java
public interface NotificadorSistemaB {
  void envia(Midia midia);
}

public class NotificadorSistemaBImpl implements NotificadorSistemaB {
  // go little horse, go!
}

class ProcessadorMidiaNoSistemaA {
  private NotificadorSistemaB notificador;

  public ProcessaMidiaNoSistemaA(NotificadorSistemaB notificador) {
    this.notificador = notificador;
  }

  void processarMidia(Midia midia) {
    persisteMidiaLocalmente(midia);
    notificador.envia(midia);
  }
}
```

Com este novo arranjo, um teste para a classe ProcessaMidiaNoSistemaA pode injetar um mock,
via construtor, tornando:

1. a classe verificável - o método processarMidia invocou o notificador?
2. implementação independente do sistema externo, injetando um notificador que não executasse
a lógica externa.


Do ponto de vista de implementação, DI oferece uma forma de contornar aquilo que pode ser considerado
um defeito na linguagem Java. Foi necessário criar:

1. uma interface,
2. uma implementação com a lógica de integração daquela interface, e
3. um construtor, para tornar a classe testável.

Linguagens como Ruby, por sua vez, oferecem meios para tornar o design mais testável, sem a necessidade
da aplicação de DI:

``` java
class ProcessadorMidiaNoSistemaA
  def processar_midia(midia)
    persisteMidiaLocalmente(midia)

    notificador = NotificadorSistemaB.new
    notificador.envia(midia)
  end
end
```

Um teste no RSpec poderia ser assim escrito:

``` ruby
describe ProcessadorMidiaNoSistemaA do
  subject { ProcessadorMidiaNoSistemaA.new }
  let(:midia) { create :midia }

  it "notifica sistema B" do
    NotificadorSistemaB.any_instance.should_receive(:envia).with(midia)
    subject.processar_midia(midia)
  end
end
```

Ruby, portanto, não é dependente de Dependency Injection quanto Java o é.

Entretanto, isto não invalida por completo a aplicação do pattern DI em alguns
cenários, caso o programador deseje desenhar construtores mais rígidos em suas
classes. Neste caso, porém, trata-se de decisão do programador, não de contornar
uma limitação da linguagem.


### Referências Bibliográficas

* [Dependency Injection is not a virtue](http://david.heinemeierhansson.com/2012/dependency-injection-is-not-a-virtue.html)
* [Are Design Patterns Missing Language Features?](http://c2.com/cgi/wiki?AreDesignPatternsMissingLanguageFeatures)
* [Rethinking Design Patterns](http://www.codinghorror.com/blog/2007/07/rethinking-design-patterns.html)
