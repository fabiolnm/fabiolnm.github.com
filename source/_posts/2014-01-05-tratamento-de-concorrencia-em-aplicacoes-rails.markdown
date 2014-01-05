---
published: true
layout: post
title: "Tratamento de Concorrência em Aplicações Rails"
date: 2014-01-05 03:32
comments: true
categories:
 - Rails
---

Em aplicações web, onde transações podem ocorrer simultaneamente, é necessário estar atento às situações
onde a concorrência pode levar à violação da integridade dos dados da aplicação. Considere por exemplo
uma aplicação que gerencia créditos e débitos, e que armazena o saldo pré-computado de uma conta.

<!-- more -->

Este problema apresenta um cenário de condição de corrida: caso duas transações ocorram simultaneamente,
o saldo pode ser atualizado de forma indevida.

Suponhamos uma conta cujo saldo seja $0, e que ocorra uma transação de crédito (C) de $100, simultaneamente
com um débito (D) de $40. Sejam Ti os tempos em que os eventos ocorrem, T0 representando um estado anterior
às transações.

<table class="concurrency_events">
<tr>
  <th></th>
  <th>C</th>
  <th>D</th>
  <th></th>
</tr>
<tr>
  <th>T0</th>
  <td>$0</td>
  <td>$0</td>
  <td>ambas as transações leêm o saldo inicial como $0</td>
</tr>
<tr>
  <th>T1</th>
  <td>$100</td>
  <td>$0</td>
  <td>a transação de crédito atualiza o saldo um pouco antes, assumindo o valor $100 naquela thread</td>
</tr>
<tr>
  <th>T2</th>
  <td>$100</td>
  <td>-$40</td>
  <td>a transação de débito atualiza o saldo um pouco depois, e o saldo assume valor -$40</td>
</tr>
<tr>
  <th>T3</th>
  <td colspan="2">$100 ou -$40</td>
  <td>o resultado da operação é igual ao da última transação executada</td>
</tr>
</table>
<br/>

Na verdade, o que se deseja é que o saldo final seja $60.

Uma forma possível de reproduzir condições de corrida em aplicações Rails é adicionar um teste de integração:

``` ruby
require "test_helper"

describe "UpdateBalance Integration Test" do
  def credit(balance, value)
    post_via_redirect transactions_path,
      balance_id: balance.id, value: value, type: 'Credit'
    ActiveRecord::Base.connection.reconnect!                # helps to avoid database lock timeouts between threads
  end

  def debit(balance, value)
    post_via_redirect transactions_path,
      balance_id: balance.id, value: value, type: 'Debit'   # helps to avoid database lock timeouts between threads
    ActiveRecord::Base.connection.reconnect!
  end

  it "handles race conditions" do
    n = 5
    n.times.collect { |i|
      balance = balances "balance_#{i}"
      [
        Thread.new { credit balance, 100 },
        Thread.new { debit balance, 40 }
      ].each {|t| t.join }
      ActiveRecord::Base.connection.reconnect!              # helps to avoid database lock timeouts
      balance.reload.value
    }.must_equal [60] * n
  end
end
```

``` bash
  1) Failure:
UpdateBalance Integration Test#test_0001_handles race conditions [/Users/fabio/workspace/miranti/LockingApp/test/integration/update_balance_test.rb:20]:
Expected: [60, 60, 60, 60, 60]
  Actual: [-40, 100, -40, 100, 100]
```

É necessário, portanto, adicionar mecanismos para impedir que o saldo seja atualizado simultaneamente por diferentes transações.

### Optimistic Locking

Para habilitar esta abordagem, basta adicionar uma coluna (```lock_version```, por default)
que é utilizada para 'versionar' um registro no banco. Cada vez que o registro é atualizado,
a coluna é incrementada.

<table class="concurrency_events">
<tr>
  <th></th>
  <th>Versão C</th>
  <th>C</th>
  <th>Versão D</th>
  <th>D</th>
  <th></th>
</tr>
<tr>
  <th>T0</th>
  <td>5</td>
  <td>$0</td>
  <td>5</td>
  <td>$0</td>
  <td>ambas as transações leêm o saldo inicial como $0 e versão 5</td>
</tr>
<tr>
  <th>T1</th>
  <td>6</td>
  <td>$100</td>
  <td>5</td>
  <td>$0</td>
  <td>a transação de crédito atualiza o saldo e incrementa a versão</td>
</tr>
<tr>
  <th>T2</th>
  <td>6</td>
  <td>$100</td>
  <td colspan="2" style="color: red; font-weight: bold">Stale</td>
  <td>
    quando tenta-se criar o débito, a aplicação percebe que a versão presente no banco já é mais atual
    que a lida inicialmente pela thread, e lança uma exceção <code>ActiveRecord::StaleObjectError</code>
  </td>
</tr>
<tr>
  <th>T3</th>
  <td>6</td>
  <td colspan="3">$100</td>
  <td>Débito não executado</td>
</tr>
</table>
<br />

O tratamento do erro fica a cargo da aplicação.

Pode-se adotar uma abordagem 'relaxada', apresentando uma mensagem de erro para o usuário, solicitando
que repita/confirme a transação:

``` ruby
  it "handles race conditions raising error" do
    n = 5
    # disparar o erro é o comportamento esperado
    assert_raise ActiveRecord::StaleObjectError do
      n.times.collect { |i|
        balance = balances "balance_#{i}"
        [
          Thread.new { credit balance, 100 },
          Thread.new { debit balance, 40 }
        ].each {|t| t.join }
        ActiveRecord::Base.connection.reconnect!
        balance.reload.value #.must_equal 60
        # Balance.find(balance.id).value
      }.must_equal [60] * n
    end
  end
```

Ou uma abordagem resiliente, obtendo a versão atualizada do saldo e operando sobre este a segunda transação:

```
  # POST /transactions
  def create
    factor    = params[:type] == 'Debit' ? -1 : 1
    @value    = factor * params[:value].to_f
    @balance  = Balance.find params[:balance_id]

    # _version, _value = @balance.lock_version, @balance.value

    success   = begin
                  create_transaction
                rescue ActiveRecord::StaleObjectError   # recover and retry
                  Balance.connection.reconnect!
                  @balance.reload
                  create_transaction
                end

    # log.debug "#{@balance.id} (v#{_version} -> v#{@balance.lock_version}): $#{_value} -> $#{@balance.value}"

    if success
      redirect_to @transaction, notice: 'Transaction was successfully created.'
    else
      render action: 'new'
    end
  end

  private
    def create_transaction
      ActiveRecord::Base.transaction do
        @transaction = Transaction.create balance: @balance, value: @value
        if @transaction.valid?
          @balance.value += @transaction.value
          @balance.save
        end
      end
    end
```

``` bash
# Running tests:
.
```

### Pessimistic Locking

Na abordagem pessimista, um determinado registro fica bloqueado, a nível de banco de dados,
(por exemplo, ```SELECT ... FOR UPDATE```) impedindo que seja alterado por uma transação concorrente.

Existem 3 formas de habilitar esta trava:

a) Diretamente na query:
``` ruby
  Balance.find(id, lock: true)  # deprecated
```

b) No objeto:
``` ruby
  balance.lock!  # o objeto é recarregado, aplicando a trava
  # some stuff...
  balance.save!
```
c) Passando um bloco:
``` ruby
  balance.with_lock do
    # some stuff
  end
```

A implementação do controller ficaria mais simples (nenhuma modificação necessária no teste):

``` ruby
  # POST /transactions
  def create
    factor    = params[:type] == 'Debit' ? -1 : 1
    @value    = factor * params[:value].to_f
    @balance  = Balance.find params[:balance_id]

    if create_transaction
      redirect_to @transaction, notice: 'Transaction was successfully created.'
    else
      render action: 'new'
    end
  end

  private
    def create_transaction
      ActiveRecord::Base.transaction do
        @transaction = Transaction.create balance: @balance, value: @value
        if @transaction.valid?
          @balance.with_lock do
            @balance.value += @transaction.value
            @balance.save
          end
        end
      end
    end
```

### Considerações Finais

Apesar de mais simples, a abordagem pessimista aplica travas no banco de dados, sendo suscetível a situações
de deadlock, ou seja, duas transações bloquearem uma à outra, caso criem dependência circular entre os
registros que elas travam. Além disso, a implementação da trava depende do banco de dados utilizado.

Na abordagem otimista, nenhum registro é travado (não há risco de deadlock), mas adiciona complexidade à
aplicação, dado que é necessário adicionar o tratamento para quando se tenta atualizar um objeto defasado (stale).
Todo o mecanismo de locking é feito pela aplicação, independente do banco de dados utilizado.

### Referências Bibliográficas

* [Concurrency Control](http://en.wikipedia.org/wiki/Concurrency_control)
* [Optimistic Locking](http://api.rubyonrails.org/classes/ActiveRecord/Locking/Optimistic.html)
* [Pessimistic Locking](http://api.rubyonrails.org/classes/ActiveRecord/Locking/Pessimistic.html)
