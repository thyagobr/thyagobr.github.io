---
layout: post
title: "Direto da Fonte: Associações no Rails"
date:   2017-10-12 09:58:45 -0300
categories: direto-da-fonte
---

O Rails nos traz muitos métodos mágicos que parecem fazer tudo o que queremos com muito pouco esforço de nossa parte. Dentre estas "mágicas" estão os métodos de associação, como **has_one**, **has_many**, **belongs_to** e o **has_and_belongs_to_many**.

Mas o que exatamente acontece quando estabelecemos uma associação num modelo do Rails?

(Este texto não pretende ser exaustivo em todos os aspectos realizados na geração de associações. Iremos caminhar pelo código de maneira casual. Eu vou deixar links para os arquivos no github, caso você tenha interesse em se aprofundar em algum ponto ou outro, à sua conveniência)

Feito este belíssimo *disclaimer*, vamos lá:

{% highlight ruby %}
class Company
 has_many :users
end

class User
  belongs_to :company
end
{% endhighlight %}

**has_many** aqui parece ser um método que recebe o argumento **:users**; o mesmo que **has_many(:users)** chamado no **self** atual, ou seja:

{% highlight ruby %}
class Company
 self.has_many(:users)
end
{% endhighlight %}

Quem é **self** neste contexto? O Ruby nos dá algumas regrinhas pra sempre saber quem é self em um determinado ponto do código, mas os detalhes vão ficar pra uma outra postagem. Neste caso, o self é a classe **Company**. Ou seja:

```
2.4.0 :013 > Company.respond_to? :has_many                                                                            
=> true
```

Mas como esse método foi parar aí?

O mais sensato parece ser ele ter sido incluído quando herdamos de **ActiveRecord::Base**. Esta classe carrega várias outras classes; uma delas é a **Associations**. Este arquivo define, dentro da classe **ActiveRecord::Associations::ClassMethods**, o método **has_many**:

{% highlight ruby %}
def has_many(name, scope = nil, **options, &extension)
  reflection = Builder::HasMany.build(self, name, scope, options, &extension)
  Reflection.add_reflection self, name, reflection
end
{% endhighlight %}

(<https://github.com/rails/rails/blob/master/activerecord/lib/active_record/associations.rb#L1403>)

O **has_many** recebe, dentre outros argumentos, o *name* - no nosso caso, *:users*. A primeira coisa que ele faz é passar todos os argumentos para **Builder::HasMany.build**

{% highlight ruby %}
def self.build(model, name, scope, options, &block)
  if model.dangerous_attribute_method?(name)
    raise ArgumentError, "You tried to define an association named #{name} on the model #{model.name}, but " \
                         "this will conflict with a method #{name} already defined by Active Record. " \
                         "Please choose a different association name."
  end

  extension = define_extensions model, name, &block
  reflection = create_reflection model, name, scope, options, extension
  define_accessors model, reflection
  define_callbacks model, reflection
  define_validations model, reflection
  reflection
end
{% endhighlight %}

(<https://github.com/rails/rails/blob/master/activerecord/lib/active_record/associations/builder/association.rb#L23>)

Primeiro este método tenta definir extensões, mas nós não passamos nenhuma para ser criada. Depois, cria a reflexão (reflection). O método para criar a reflexão está logo abaixo do que estamos analisando. Ele tenta criar escopos, o que também não solicitamos, e passa para criar de fato a reflexão no método **ActiveRecord::Reflection.create**.

Perceba que o primeiro parâmetro passado é “macro”, mas não temos esta variável definida em lugar nenhum. Ela será definida por sub-classes, não por nós ou por esta classe. Mais especificamente:

{% highlight ruby %}
module ActiveRecord::Associations::Builder # :nodoc:
  class HasMany < CollectionAssociation #:nodoc:
    def self.macro
      :has_many
    end
    ...
  end
end
{% endhighlight %}

Resolvida a questão do macro, chegamos no método ActiveRecord::Reflection.create:

{% highlight ruby %}
def self.create(macro, name, scope, options, ar)
  klass = \
    case macro
    when :composed_of
      AggregateReflection
    when :has_many
      HasManyReflection
    when :has_one
      HasOneReflection
    when :belongs_to
      BelongsToReflection
    else
      raise "Unsupported Macro: #{macro}"
    end

  reflection = klass.new(name, scope, options, ar)
  options[:through] ? ThroughReflection.new(reflection) : reflection
end
{% endhighlight %}

(<https://github.com/rails/rails/blob/master/activerecord/lib/active_record/reflection.rb#L17>)

E o que ele faz é selecionar a classe de reflexão adeqüada de acordo com um case/when em cima do macro (é, eu sei). Feito isso, ele instancia a classe de reflection, que está definida no mesmo arquivo, [na linha 687](https://github.com/rails/rails/blob/master/activerecord/lib/active_record/reflection.rb#L687). Mas o **initialize**, que é o que nos interessa no **klass.new**, está na classe herdada **AssociationReflection**, definido [na linha 426](https://github.com/rails/rails/blob/master/activerecord/lib/active_record/reflection.rb#L426). O **initialize** apenas seta algumas variáveis necessárias para realizar a reflexão.

Depois de criada e retornada a classe de reflexão, o **ActiveRecord::Reflection.create** só verifica se a opção *through* foi passada e, em caso positivo, encapsula a classe de reflexão em outra classe de reflexão específica para relacionamentos *through*. Não é nosso caso, simplesmente retornamos nosso objeto de volta para o método que o chamou, o **Build::HasMany.build**

Finalmente temos nossa classe de reflexão criada. O próximo passo é o método **define_accessors**. Aqui é onde são definidos os getters e setters para os relacionamentos - no nosso caso, os métodos **company.users** e **company.users=**. Eis os métodos interessantes ao caso:

{% highlight ruby %}
def self.define_accessors(model, reflection)
  mixin = model.generated_association_methods
  name = reflection.name
  define_readers(mixin, name)
  define_writers(mixin, name)
end

def self.define_readers(mixin, name)
  mixin.class_eval <<-CODE, __FILE__, __LINE__ + 1
    def #{name}(*args)
      association(:#{name}).reader(*args)
    end
  CODE
end

def self.define_writers(mixin, name)
  mixin.class_eval <<-CODE, __FILE__, __LINE__ + 1
    def #{name}=(value)
      association(:#{name}).writer(value)
    end
  CODE
end
{% endhighlight %}


O método está definido [na linha 98](https://github.com/rails/rails/blob/master/activerecord/lib/active_record/associations/builder/association.rb#L23), e chama dois outros métodos logo abaixo. Olhando estes dois outros métodos, vemos que os getters e setters são *syntatic sugars*, ou seja, são uma forma mais elegante de enviar uma mensagem mais prolixa.

{% highlight ruby %}
c = Company.new
c.association(:users).reader # é o mesmo que c.users
c.association(:users).writer(users) # é o mesmo que c.users = users
{% endhighlight %}

Finalmente, o último passo é adicionar a reflexão numa variável que serve como cache para todas as reflexões do modelo.

Existem vários detalhes e caminhos alternativos que não foram abordados neste texto. Eu espero ter colaborado com guias gerais sobre como pesquisá-los, e recomendo a todos fazê-lo. Alguns exemplos: entender o que acontece em relacionamentos *through*, a criação de scopes e e a criação de extensions, para citar alguns.

===

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
