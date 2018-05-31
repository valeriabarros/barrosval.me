---
layout: post
title: jekyll + SSH + Travis
excerpt: ""
categories: [tutorial]
modified: 2018-05-30
comments: true
---

> Esse tutorial faz parte da série de tutoriais que não achei em português, então vos apresento:

### Um guia de como subir seu site em jekyll para um servidor de hospedagem com acesso SSH usando Travis

Fazia tempo que eu queria dar uma atualizada nas minhas plataformas, e como hoje em dia existem 10000 milhões de possibilidades, aproveitei a oportunidade para testar um conceito que ainda não tinha entendido: **sites estáticos**.
Testei [hugo](https://gohugo.io/), mas acabei decidindo pelo Jekyll.

### Por que Integração Contínua?
 > Integração Contínua permite que você publique seu site gerado pelo Jekyll com confiança, automatizando os processos de garantia de qualidade e implantação.

<div style="overflow:hidden;padding:10px 90px;">
  <img style="float: left" src="https://media1.tenor.com/images/20fe04f27ada64dc461d09def5b003ca/tenor.gif?itemid=5540972">
    <li> push e sair correndo</li>
    <li> trabalho só uma vez</li>
    <li> robôs!</li>
    <li> menos perda de tempo</li>
    <li> mais alegria</li>
    <li> você merece! </li>
</div>

### Bora pra action!
 - Sem papo furado:
    - Crie uma conta no travis-ci.org 
    - Vincule com seu github
    - Selecione o repositório que você deseja configurar.

Na sua máquina, você precisa criar um arquivo chamado `.travis.yml` dentro do seu repositório. Aqui temos um exemplo padrão de como a sintaxe funciona:
 {% highlight yml %}
   language: ruby
   rvm:
    - 2.4 #versao do ruby
  env:
    global:
    - NOKOGIRI_USE_SYSTEM_LIBRARIES=true
  branches:
    only:
    - master
  install: gem install jekyll
  script:
    - jekyll build # você pode adicionar quantos scripts você quiser, para teste, validação e etc.
  {% endhighlight %}

---

A partir daqui, vamos trabalhar com as configurações para permitir que o Travis acesse o seu servidor e o atualize com seu site!
Para isso, você precisa instalar o serviço do travis para linha de comando na sua máquina.
> `brew install travis`

#### SSH
SSH nada mais é que um protocolo de acesso virtual de uma máquina para outra, por exemplo, da sua máquina para o servidor de hospedagem.
É exatamente isso que vamos configurar, o acesso SSH entre o travis e o servidor.

Abra seu terminal no diretório em que se encontra seu blog e execute o seguinte comando:

{% highlight bash %}
  ssh-keygen -t rsa -b 4096 -C 'build@travis-ci.org' -f ./deploy_rsa
{% endhighlight %}
Esse comando irá gerar suas chaves SSH privada e pública, `deploy_rsa` e `deploy_rsa.pub` respectivamente, e são elas que irão garantir a conexão.

Novamente no seu terminal:
{% highlight bash %}
  travis encrypt-file deploy_rsa --add
{% endhighlight %}

Caso seja seu primeiro acesso ao travis na linha de comando, você precisará entrar com seus dados de login (aqueles que você criou lá em cima!). Esse comando irá encriptar (essa palavra existe!) sua chave privada, gerando o arquivo `deploy_rsa.enc` e adicionando os dados da chanve no seu arquivo `.travis.yml`.


Com nossas chaves lindas geradas, criptografadas e seguras, chegou a hora de pedir acesso ao servidor de hospedagem:
{% highlight bash %}
    ssh-copy-id -i deploy_rsa.pub user@hostname
{% endhighlight %}
Esse comando permite que o servidor de hospedagem saiba que nossa identidade, batendo as chaves. Se tudo funcionar, nossa conexão será estabelecida sem uso de senha. 

> Para saber se tudo está funcionando, faça o teste de conexão com o servidor. Se você precisar utilizar sua senha novamente, significa que algo está errado. 

Caso seu servidor seja hospedado pela Dreamhost, você pode ter problemas com a permissão para criação de acessos remotos. Uma opção para resolver o problema é, ao invés de `ssh-copy-id`:
{% highlight bash %}
    scp deploy_rsa.pub username@hostname:~/pasta-do-seu-site
{% endhighlight %}

Agora a única chave que você precisa está encriptada,  então é seguro remover suas outras chaves:

{% highlight bash %}
    rm -f deploy_rsa deploy_rsa.pub
{% endhighlight %}

Ainda no terminal, precisamos dizer para o Travis de maneira segura em qual servidor e diretório o site deverá subir:
{% highlight bash %}
    travis encrypt DEPLOY_DIRECTORY=/path/to/folder --add
    travis encrypt DEPLOY_HOST=hostname_ou_IP --add
    travis encrypt DEPLOY_USER=username --add
{% endhighlight %}

> Esse comando irá adicionar três linhas no seu arquivo de configuração em `env  global`, iniciando com `- secure`, que nada mais é que seus dados de deploy criptografados :-)

---

Abra seu arquivo `.travis.yml` e você irá notar que algumas informações foram adicionadas. Vamos fazer algumas mudanças:

- Na sessão `script` você verá alguns comandos relacionados ao SSH. A primeira mudança é mover essas informações para `before_deploy`.

- Na sessão `before_deploy`, adicione a seguinte linha:
`- echo -e "Host $DEPLOY_HOST\n\tStrictHostKeyChecking no\n" >> ~/.ssh/config`

Agora o `before_deploy` deve estar assim: 
{% highlight yml %}
    before_deploy:
    - openssl aes-256-cbc -K $encrypted_hash_key -iv $encrypted_hash_iv
      -in deploy_rsa.enc -out deploy_rsa -d
    - eval "$(ssh-agent -s)"
    - chmod 600 deploy_rsa
    - ssh-add deploy_rsa
    - echo -e "Host $DEPLOY_HOST\n\tStrictHostKeyChecking no\n" >> ~/.ssh/config
{% endhighlight %}

Adicione o seguinte código ao final do seu arquivo:

{% highlight yml %}
    deploy:
      provider: script
      skip_cleanup: true
      script: rsync -r --quiet --delete-after _site/* $DEPLOY_USER@$DEPLOY_HOST:$DEPLOY_DIRECTORY
      on:
        branch: master
{% endhighlight %}

O `skip_cleanup` informa ao travis para não excluir a build antes de completar a execução, enquanto o `--delete-after` limpa o diretório no servidor de hospedagem antes de subir o novo site, para garantir integridade.

Acabamos a parte fácil. Seu arquivo de configuração agora está pronto, e deve estar parecido com esse aqui:
{% highlight yml %}
    language: ruby
    rvm:
    - 2.4
    env:
      global:
      - NOKOGIRI_USE_SYSTEM_LIBRARIES=true
      - secure: hash
      - secure: hash
      - secure: hash
    branches:
      only:
      - stable #yourBranch
    install: gem install jekyll #install jekyll
    script:
    - jekyll build #build jekyll
    before_deploy:
    - openssl aes-256-cbc -K $encrypted_hash_key -iv $encrypted_hash_iv
      -in deploy_rsa.enc -out deploy_rsa -d
    - eval "$(ssh-agent -s)"
    - chmod 600 deploy_rsa
    - ssh-add deploy_rsa
    - echo -e "Host $DEPLOY_HOST\n\tStrictHostKeyChecking no\n" >> ~/.ssh/config
    sudo: false
    deploy:
      provider: script
      skip_cleanup: true
      script: rsync -r --quiet --delete-after _site/* $DEPLOY_USER@$DEPLOY_HOST:$DEPLOY_DIRECTORY
      on:
        branch: stable
{% endhighlight %}

---
### Acabou!

Agora é a hora da verdade. Para testar, faça um push no seu repositório (na branch que você indicou na sessão `branches`). Automaticamente o Travis irá reconhecer a trigger, e iniciar a tentativa de deploy. Você pode acompanhar a evolução no [travis](https://travis-ci.org/).

Tudo bem se não funcionar de primeira! Dê uma olhada na mensagem de erro e tente novamente!
Eu não sou de me emocionar muito com a vida nerd, mas sinceramente, foi um momento de MUITA alegria quando consegui pela primeira vez :D

E se você chegou até aqui, saiba:
![a](https://media1.tenor.com/images/e9f13b9ce931788872db099c85f89c94/tenor.gif?itemid=5701772)

Happy coding!