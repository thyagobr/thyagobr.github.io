---
layout: post
title: "Respondendo a diferentes formatos de requisição no Rails"
date: 2017-04-10 05:01:00 -0300
categories: rails iniciante
---
(Aqui iremos tratar sobre como fazer um controller Rails tradicional responder, quando necessário, com JSON)

Vamos supor que nós temos um app para listar informações sobre músicas, onde o usuário pode ver todas as músicas cadastradas no sistema.

O controller que responde esta requisição está da seguinte forma:

{% highlight ruby %}
class SongsController < ApplicationController
  def index
    @songs = Song.all
  end
end
{% endhighlight %}

O usuário navega para `/songs`, o Rails chama o `SongsController#index` que carrega todas as músicas e as envia para a view em `views/songs/index.html.erb`. O app foi um sucesso e o CEO da startup fechou uma parceria para vender o seu gosto músical para empresas que vão tentar usar os dados pra te vender coisas. Ou seja, nova feature: fazer o `/songs` retornar JSON, quando solicitado.

Como fazer para um mesmo endpoint retornar às vezes JSON, às vezes HTML, dependendo do que o usuário quiser?

Na verdade, este endpoint já faz isso. O Rails - seguindo o lema de "convenção antes de configuração" - chama automáticamente um [render][], quando nenhum é chamado pelo desenvolvedor. O `render` é responsável por renderizar o conteúdo, ou seja: formatar o conteúdo de acordo com a saída selecionada. Mas como o Rails sabe pra qual formato - XML, JSON, HTML, CSV, etc. - eu quero a minha saída?

Ele não pode fazer mágica: precisamos informar de alguma maneira. Quando você navega para `localhost:3000/songs`, o seu navegador preenche um cabeçalho HTTP com algumas informações para o servidor, como:

```
Accept:text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8
Accept-Encoding:gzip, deflate, sdch, br
Accept-Language:en-GB,en-US;q=0.8,en;q=0.6
Connection:keep-alive
Cookie:csrftoken=drune4i73GrT2OU5GHr9M1Mqfl8maG7J; dwf_section_edit=True; dwf_sg_task_completion=False
DNT:1
Host:developer.mozilla.org
Referer:https://www.google.com.br/
Upgrade-Insecure-Requests:1
User-Agent:Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/56.0.2924.87 Safari/537.36
```

O Rails lê o `Accept` deste cabeçalho e decide-se rapidamente por renderizar o conteúdo em `text/html`.

O Rails atualmente já traz a gem `jbuilder` como default. Usando as versões mais recentes (Rails 4+, em abril de 2017) do Rails, encontraremos o seguinte arquivo no diretório `app/views/baskets/index.json.jbuilder`:

{% highlight ruby %}
json.array!(@songs) do |song|
  json.extract! song, :id, :title, :artist_name
  json.url song_url(song, format: :json)
end
{% endhighlight %}

Esse é o arquivo que "renderiza" a saída JSON, usando o jbuilder; é a view do JSON, se preferir (tanto que reside junto das outras views). Ele é invocado pelo rails quando a renderização é para JSON, assim como o `index.html.erb` é invocado quando a saída é HTML. Esta "view" decide quais atributos serão exibidos e de que forma. Várias operações podem ser feitas aqui para modelar a saída, como criar campos que sintetizem outros campos, etc. Age mais como um "Presenter", mas isso é um detalhe.

Até aqui sabemos que o Rails já responde a vários formatos de saída, mas que apenas para HTML e JSON as últimas versões trazem um renderizador pronto. E o que acontece se tentarmos acessar um formato para o qual não temos uma camada de apresentação configurada para ser usada pelo renderizador? Por exemplo, se deletarmos o `index.json.jbuilder` (ou se tentarmos acessar `localhost:3000/songs.xml`), o que acontece? Exato, um erro. Mais especificamente, o famoso "Missing template", que significa que a ação foi processada, mas não temos um template adeqüado para mostrar os resultados no formato desejado - o que não é necessariamente verdade. No caso do XML, e de todos os outros formatos exceto HTML e JSON, o erro correto seria dizer que nós não os reconhecemos. Então, vamos liberar apenas os dois formatos com que queremos trabalhar:

{% highlight ruby %}
class SongsController < ApplicationController
  def index
    @songs = Song.all
    respond_to :html, :json
    end
  end
end
{% endhighlight %}

Tudo continua da mesma forma que estava antes, mas agora nós só aceitamos HTML e JSON. Da forma que está, usando gems como a já mencionada `jbuilder` ou qualquer outra de mesma função, como a `ActiveModel::Serializer`, a resposta JSON está implementada.

Caso você não precise fazer nenhuma manipulação nos dados do objeto e deseje incluir todos os campos na saída JSON, não precisa instalar uma gem só pra isso. Podemos passar um block para o `respond_to` para informarmos manualmente como deve ser a saída, evitando assim que o Rails precise perguntar a uma gem.

Vamos alterar o `SongsController#index`:

{% highlight ruby %}
class SongsController < ApplicationController
  def index
    @songs = Song.all

    respond_to do |format|
      format.html
      format.json { render json: @songs }
    end
  end
end
{% endhighlight %}

O `format.html` sem nenhum parâmetro faz exatamente a mesma coisa que fazia antes: repassa a responsabilidade para o template padrão (ou seja, `index.html.erb`). No caso do `format.json`, nós estamos chamando o `#render` e informado os parâmetros manualmente (o framework se encarrega de chamar o `#to_json` no `@songs`). Este bloco pode ser especificado de [várias formas interessantes][api-dock-respond_to]{:target="_blank"}, mas vamos manter este post introdutório.

Ah, não custa lembrar: caso você não vá precisar servir HTML no seu projeto, você pode [criar o seu app Rails no modo API][create-rails-api]{:target="_blank"}


Referências:

 * https://apidock.com/rails/ActionController/MimeResponds/respond_to
 * http://www.justinweiss.com/articles/respond-to-without-all-the-pain/

[api-dock-respond_to]: https://apidock.com/rails/ActionController/MimeResponds/respond_to
[create-rails-api]: #
