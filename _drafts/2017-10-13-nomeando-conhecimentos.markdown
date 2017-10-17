---
layout: post
title: "Nomeando conhecimentos"
date: 2017-10-13 12:57 -0300
categories: pensando-em-código
---
"Só existem duas difíceis nas ciências da computação: invalidação de cache e dar nome às coisas" é uma frase do Phil Karlton que é conhecida entre programadores em todo o mundo. E ela possui aquele atributo de clássicos: quanto mais vivemos experiências, mais ela parece se aprofundar em significados.

Quando eu a ouvi pela primeira vez na graduação eu tive a minha primeira interpretação, com uma noção puramente estética. Dar nome às coisas realmente é difícil: um novo personagem em um jogo ou texto, um bichinho de estimação, um filho ou filha, um novo projeto, etc. Estudos e mais estudos são feitos pelos publicitários e marqueteiros das mais variadas especialidades para determinar como fazer nomes grudarem nas cabeças dos potenciais consumidores. Chega a ser paralisante em alguns casos.

Mas será que é só isso que a frase quer (ou pode) dizer?

Aqui vale a pena pensar um pouco: o que precisamos nomear quando estamos construindo sistemas? Se você estiver disposto a fazer um teste, eu recomendaria gaster uns 2 minutos e escrever em algum lugar coisas que um programador precisa nomear num sistema. Talvez, alguns dos seguintes ítens estarão na sua lista:

* projetos
* classes
* métodos

Imaginemos a seguinte situação: quando um evento for criado em um sistema precisamos criar um alerta relativo àquele evento e dispará-lo para web através de um canal websocket. O evento é criado por um endpoint na nossa API. Então, teremos, por exemplo:

{% highlight ruby %}
class Api::V1::EventsController < Api::V1::ApiController
  def create
    event = Event.new(event_params)
    if event.save
      alert = Alert.new(event: event)
      alert.created_by = current_user
      alert.company = current_company
      alert.save
      ActionCable.server.broadcast("alerts_channel_#{current_company.id}", AlertSerializer.new(alert).serializable_hash)
      render json: event
    else
      render json: event.errors
    end
  end

  private

  def event_params
    params.fetch(:event, {}).permit(:id, :event_type, :priority, :location_id)
  end
end
{% endhighlight %}

O que está acontecendo nesse controller? Ele está:

1. recebendo os parâmetros do evento
2. criando um evento
3. criando um alerta
4. enviando o alerta via broadcast para o websocket
5. renderizando a resposta

Parece coisa demais pra que o controller resolva sozinho. E, além do mais, nós não estamos nomeando estes conhecimentos que listamos. Que o controller serve para receber um request, delegar tarefas e renderizar a response todos nós sabemos: esta é a definição do que é um controller. Perceba que o recebimento de parâmetros e a renderização estão relativamente bem nomeados, através do `event_params`, definido no escopo privado, e o método `render`, respectivamente. Nos outros casos, é questionável.

{% highlight ruby %}
def generate_alert(event)
  alert = Alert.new(event: event)
  alert.created_by = current_user
  alert.company = current_company
  ActionCable.server.broadcast("alerts_channel_#{current_company.id}", AlertSerializer.new(alert).serializable_hash)
  alert
end
{% endhighlight %}
