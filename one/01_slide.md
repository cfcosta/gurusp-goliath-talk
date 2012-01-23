!SLIDE
# Programação Web Assíncrona com Goliath #

!SLIDE bullets
# Cainã Costa #
* @sryche

!SLIDE center
# Hoje é Sabagato! #
![Hoje é Caturday!](caturday.jpg)

!SLIDE bullets incremental
# Disclaimer #
* Minha primeira palestra na vida.

!SLIDE
# Eventos são maneiros, mas... #

* Eles são difíceis de testar
* Eles são difíceis de acompanhar
* Eles não são o que estamos acostumados...

!SLIDE bullets incremental

# Podemos simplesmente nos acostumar com o modelo evented! #

* Mas... Eu tenho preguiça!
* Sério, é só por isso.

!SLIDE
# Existe outra solução? #

!SLIDE

# Fibers! #

!SLIDE
# Mas primeiro, um histórico #

!SLIDE center
# Global Interpreter Lock #
![Global Interpreter Lock](ruby-gil.png)

!SLIDE center
# Fibers vs Threads #
![Fibers vs Threads](fibers-vs-threads.png)

!SLIDE
# Beleza, mas pra que serve? #

!SLIDE

# Podemos transformar isso... #

    @@@ ruby
    require 'eventmachine'
    require 'em-http'

    EM.run do
      url = "http://jsonip.com"
      http = EM::HttpRequest.new(url).get timeout: 10
      http.callback do |data|
        url = "http://localhost:9292"
        http = EM::HttpRequest.new(url).post timeout: 10,
                                                    body: {ip: data.response}

        http.callback do |data|
          puts data.response

          EM.stop
        end
      end
    end

!SLIDE

# Nisso! Look, ma! No callbacks! #

    @@@ ruby
    require 'fiber'
    require 'eventmachine'
    require 'em-http'

    def async_fetch(url, method = :get, params = {timeout: 10})
      f = Fiber.current
      http = EM::HttpRequest.new(url).send(method, params)
      http.callback { f.resume(http) }

      return Fiber.yield
    end

    EM.run do
      Fiber.new{
        puts "Setting up HTTP request #1"
        data = async_fetch('http://jsonip.com/')

        post = async_fetch('http://localhost:9292/',
                           :post, timeout: 10, body: {ip: data.response})

        puts post.response

        EM.stop
      }.resume
    end

!SLIDE

# EventMachine::Synchrony

!SLIDE

# Lembra do nosso exemplo? Então... #

    @@@ ruby
    EM.run do
      EM.synchrony do
        data = EM::HttpRequest.new('http://jsonip.com/').get

        post = EM::HttpRequest.new('http://localhost:9292/').
                               post(timeout: 10, body: {ip: data.response})

        puts post.response

        EM.stop
      end
    end

!SLIDE
# O EM::Synchrony cuida da parte complicada pra gente...
* Coloca o bloco inteiro dentro de uma Fiber
* Cuida do scheduling da Fiber para parar/retornar assim que um I/O blocante ocorrer

!SLIDE
# Mas e se isso fosse aplicado para o Rack? #

!SLIDE
# Goliath! #