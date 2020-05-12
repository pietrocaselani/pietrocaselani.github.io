---
layout: single
title:  "Trabalhando com Realm e RxSwift"
date:   2018-02-18 16:20:00 +0200
tags: realm rxswift learnings
categories: learnings reactive
---
Olá! Meu primeiro post aqui, e estou escrevendo devido a este tweet.

<blockquote class="twitter-tweet" data-lang="en"><p lang="en" dir="ltr">Write the article you wish you found when you googled something.<a href="https://twitter.com/chriscoyier/status/925081793576837120">https://twitter.com/chriscoyier/status/925081793576837120</a></p>&mdash; Chris Coyier (@chriscoyier)<a href="https://twitter.com/chriscoyier/status/925081793576837120">October 30, 2017</a></blockquote> <script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>

## Contexto

Estou trabalhando em um projeto pessoal para iOS faz alguns meses, e precisava de uma solução para persistir alguns dados e gostaria de integrar facilmente com RxSwift, já que meu plano é ficar observando as mudanças persistidas, enquanto atualizo os dados pela API. Depois de pesquisar algumas soluções que envolviam cache, SQL e NoSQL, fiquei entre [GRDB.swift](https://github.com/groue/GRDB.swift) e [Realm](https://github.com/realm/realm-cocoa). Ambos possuem bom suporte a Rx, mas decidi ir com Realm devido a nunca ter utilizado antes e também não queria pensar em CREATE TABLE e afins.

## Então vamos lá

Comecei fazendo um aplicativo para testes, apenas com Realm para ir me familiarizando com a ferramenta. Após, integrei com RxRealm, consegui simular o caso que queria fazer no aplicativo.

Como disse, era minha primeira experiência com Realm, mas eu tinha em mente que era uma boa abordagem manter apenas uma instância do Realm aberta e fazer todas as operações fora da main thread, mesmo que muitos recomendam e dizem que não há problemas de utilizar o Realm na main thread. Eu já tinha lido como o Realm se comporta em relação a threads, e pensei que seria fácil tratar isso com Rx usando os operadores `subscribeOn` e `observeOn`.


### Pensou errado

Fiz o seguinte código bem feliz e confiante

{% highlight swift %}
let realmQueue = DispatchQueue(label: "RealmQueue", qos: .background)
let realmScheduler = SerialDispatchQueueScheduler(queue: realmQueue, internalSerialQueueName: "RealmScheduler")
{% endhighlight %}

e, assim, a situação passou de felicidade e confiança para tensa e dramática quando vi essa exceção:

{% highlight plaintext %}
Terminating app due to uncaught exception ‘RLMException’, reason:
‘Can only add notification blocks from within runloops.’
{% endhighlight %}

Fui na documentação do Realm e encontrei isso na área de notificações:


> Notifications are always delivered on the thread that they were originally registered on. That thread must have a currently running run loop. If you wish to register notifications on a thread other than the main thread, you are responsible for configuring and starting a run loop on that thread if one doesn’t already exist.

Ficou claro que apenas a main thread possui um run loop, e se você quiser observar as mudanças do Realm em outra thread, é preciso criar um `run loop`.

Confesso que nunca havia mexido na API de `run loop`, então aqui eu tentei muito, e depois de dias de tentativas e falhas, cheguei em um código que me permite ter apenas uma instância do Realm aberta, e consigo realizar todas as operações fora da `main thread`.

Primeiramente foi preciso criar uma thread, e configurar um `run loop` a ela.

{% highlight swift %}
final class ThreadWithRunLoop: Thread {
  var runLoop: RunLoop!

  override func main() {
    runLoop = RunLoop.current
    runLoop.add(Port(), forMode: .commonModes)
    runLoop.run()
  }
}
{% endhighlight %}

Após, criei um `scheduler` para trabalhar com essa thread.

{% highlight swift %}
final class ThreadWithRunLoopScheduler: ImmediateSchedulerType {
  private let thread: ThreadWithRunLoop

  init(name: String) {
    thread = ThreadWithRunLoop()
    thread.name = name
    thread.start()
  }

  func schedule<StateType>(_ state: StateType, action: @escaping (StateType) -> Disposable) -> Disposable {
    let disposable = SingleAssignmentDisposable()

    thread.runLoop.perform {
      disposable.setDisposable(action(state))
    }

    return disposable
  }
}
{% endhighlight %}

E pronto! Agora temos um run loop associado a uma thread e um scheduler que executa ações nessa thread, e dessa forma consegui fazer o app persistir e observar os dados, usando Realm e RxSwift. Para mais informações, você pode encontrar o código do aplicativo aqui.

Um ponto que vale a pena ressaltar é que o método perform que trabalha com closures só está disponível a partir do iOS 10.

Espero que esse post possa ajudar alguém, mas de qualquer forma este caso me ajudou a entender um pouco mais sobre run loop, Realm e com minha primeira contribuição aqui.
