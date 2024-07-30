---
slug: 'flask-htmx-e-alertas'
title: 'Flask, HTMX e alertas'
description: 'Como criar alertas dinâmicos com HTMX e ferramentas próprias do Flask.'
pubDate: 2024-07-30T03:00:00
tags: ['flask', 'htmx', 'python', 'tutorial']
image:
  src: 'flask_htmx.png'
  alt: 'Flask + HTMX'
---

[HTMX](https://htmx.org/) é uma biblioteca poderosa para adicionar interatividade em aplicações tradicionais. Ao adicionar atributos em elementos, permite a manipulação do DOM com requisições contendo fragmentos HTML, _event triggers_ e até WebSockets. Inspirado pelo artigo do Benoît Blanchon sobre como utilizar [HTMX com o framework de messages do Django](https://blog.benoitblanchon.fr/django-htmx-messages-framework/), resolvi adaptar essa lógica para o Flask, utilizando _triggers_ e a função utilitária `flash`.

## Boilerplate

Uma das vantagens (e desvantagens) do Flask é que podemos criar tudo em um único arquivo `app.py`, facilitando
esse breve tutorial. Os passos abaixo foram reproduzidos com Python 3.12:

```shell
python -m venv .venv --upgrade-deps
source .venv/bin/activate
echo "Flask==3.0.3" >> requirements.txt
pip install -r requirements.txt
```

Crie o arquivo principal `app.py` e faça o _boilerplate_ de uma aplicação Flask:

```python
from flask import Flask, render_template

app = Flask(__name__)


@app.route('/')
def index():
    return render_template('index.html')

```

Crie um arquivo `templates/index.html`:

```html
<!DOCTYPE html>
<html lang="pt-br">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Flask + HTMX + flash</title>
  <link rel="stylesheet" href="{{ url_for('static', filename='css/style.css') }}">
  <script type="module" src="{{ url_for('static', filename='js/main.js') }}"></script>
</head>
<body>
  <section>
    <h1>Olá mundo!</h1>
  </section>
  <div id="alerts"></div>
</body>
</html>
```

Adicione uma estilização básica em `static/css/style.css`:

```css
body {
  font-family: system-ui, sans-serif;
  max-width: 1024px;
  min-height: 100vh;
  margin: 0 auto;
  background-color: rgb(249 250 251);
  display: grid;
  place-items: center;
}

section {
  padding: 40px 20px;
  width: 480px;
  text-align: center;
}

h1 {
  font-size: 48px;
  margin: 0 0 20px 0;
}

button {
  padding: 10px 20px;
  outline: none;
  color: #fff;
  background-color: rgb(29 78 216);
  cursor: pointer;
  font-family: inherit;
  border: none;
  border-radius: 6px;
  font-size: 1em;
  font-weight: 600;
  margin: 10px 0;
}

button:hover {
  background-color: rgb(30 64 175);
}
```

E um caloroso olá do nosso arquivo `static/js/main.js`:

```js
console.log('Olá mundo 👋️')
```

No fim dessa etapa, a estrutura do projeto deve ser:

```
.
├── app.py
├── static
│   ├── js
│   │   └── main.js
│   └── css
│       └── style.css
└── templates
    └── index.html
```

Inicie o servidor de desenvolvimento com `flask run --debug`, acesse `http://localhost:5000` 
e garanta que a aplicação está funcionando corretamente.

## HTMX

Para trabalhar com a biblioteca, precisamos importar ela no nosso `templates/index.html`:

```html
<!-- dentro da tag <head> -->
  <script src="https://unpkg.com/htmx.org@2.0.1" integrity="sha384-QWGpdj554B4ETpJJC9z+ZHJcA/i59TyjxEPXiiUgN2WmTyV5OEZWCD6gQhgkdpB/" crossorigin="anonymous"></script>
<!-- </head> -->
```

E podemos fazer um protótipo de um alerta, retornando-o como HTML em uma chamada
à um endpoint. Adicione no arquivo `app.py`:

```python
import random

...

@app.route('/alert')
def alert():
    level = random.choice(['info', 'error', 'success', 'warning'])
    return f'''
    <div class="alert is-{level}">
      <strong>{level.upper()}</strong> Esse é um alerta vindo do Flask!
    </div>'''
```

E a chamada no arquivo `templates/index.html`:

```html
<body>
  <section>
    <h1>Olá mundo</h1>
    <button hx-get="{{ url_for('alert') }}" hx-target="#alerts">
      Criar um alerta aleatório
    </button>
  </section>
  <div id="alerts"></div>
<body>
```

Também vamos adicionar uma estilização para os alertas em `static/css/style.css`:

```css
/** estilos anteriores **/
#alerts {
  position: fixed;
  z-index: 10;
  left: 0;
  right: 0;
  top: 5px;
  max-width: 480px;
  margin: 0 auto;
}

.is-info {
  --alert-color: rgb(30 64 175);
  --alert-background-color: rgb(239 246 255);
}

.is-error {
  --alert-color: rgb(153 27 27);
  --alert-background-color: rgb(254 242 242);
}

.is-warning {
  --alert-color: rgb(133 77 14);
  --alert-background-color: rgb(254 252 232);
}

.is-success {
  --alert-color: rgb(22 101 52);
  --alert-background-color: rgb(240 253 244);
}

.alert {
  margin: 10px 0;
  background-color: var(--alert-background-color);
  color: var(--alert-color);
  border: 2px dashed var(--alert-color);
  border-radius: 10px;
  padding: 22px 24px;
  max-width: 480px;
}
```

Ao clicar no botão já teremos nosso alerta aleatório aparecendo no canto superior da tela. Apesar de ser próximo do que
queremos, ainda não é interativo o suficiente:
- Temos que construir o HTML no servidor
- O alerta está fixo no DOM

![Alerta estático](/images/alerta-estatico.gif)

Para adicionarmos mais interatividade, precisamos de um pouco de JavaScript e dos Event Triggers do HTMX.

## JS e Event Triggers

Ao invés de enviarmos o HTML do servidor, podemos criar ele no lado do cliente com JavaScript ao ouvir um evento no corpo
do documento. A lógica em `static/js/main.js` será:

```js
class Alert {
  #ALERT_DURATION = 3000

  constructor(category, message) {
    this.category = category
    this.message = message
  }

  #createAlertElement() {
    const alert = document.createElement('div')
    alert.className = `alert is-${this.category}`
    alert.textContent = this.message

    return alert
  }

  show(queryContainer = '#alerts') {
    const alertsContainer = document.querySelector(queryContainer)
    if (!alertsContainer) {
      console.error('Não encontrou o container de alerts!')
      return
    }

    const alert = this.#createAlertElement()
    alertsContainer.appendChild(alert)

    setTimeout(() => alert.remove(), this.#ALERT_DURATION)
  }
}

document.body.addEventListener('alerts', (event) => {
  const alerts = event.detail.value
  alerts.forEach(([category, message]) => {
    const alert = new Alert(category, message)
    alert.show()
  })
})
```

E no lado do servidor, em `app.py` nós vamos criar um trigger para o HTMX:

```python
import random
import json
from flask import Flask, render_template, make_response

...

@app.route('/alert')
def alert():
    level = random.choice(['info', 'error', 'success', 'warning'])
    response = make_response()
    response.headers['HX-Trigger'] = json.dumps({
      'alerts': [('info', 'Esse é um alerta vindo do HX-Trigger!')]
    })
    return response
```

A última modificação é retirar o atributo `hx-target` e adicionarmos o atributo
`hx-swap: none`, visto que a resposta do servidor sempre será vazia e não queremos
fazer o _swap_ do conteúdo.

```html
<!-- dentro de <section> -->
<button hx-get="{{ url_for('alert') }}" hx-swap="none">
  Criar um alerta aleatório
</button>
```

![Alerta dinâmico](/images/alerta-dinamico.gif)
## Flash e Middlewares no Flask

Com essa lógica, toda vez que precisarmos de uma notificação no cliente basta enviar uma resposta com o trigger
do HTMX, mas o processo pode se tornar repetitivo. 

Por sorte, o `Flask` vem com uma função utilitária para mostrar alertas ao usuário, o `flash`. Essa função 
salva a mensagem e a categoria dela dentro do cookie de sessão, mas podemos criar um "middleware" na resposta do 
Flask para coletar os dados salvos pelo `flash` e transformarmos em um trigger para o HTMX.

No arquivo `app.py`, adicione:

```python
import random
import json
from flask import Flask, render_template, make_response, session, flash

app = Flask(__name__)
app.secret_key = '<super-secret>'


@app.after_request
def flash_to_htmx_trigger(response):
    """Middleware que converte flashes para HX-Trigger."""
    flashes = session.pop('_flashes') if '_flashes' in session else []

    if not flashes:
        return response
  
    trigger_content = {'alerts': flashes}
    response.headers['HX-Trigger'] = json.dumps(trigger_content)

    return response

...

@app.route('/alert')
def alert():
    level = random.choice(['info', 'error', 'success', 'warning'])
    flash('Esse é um alerta vindo do HX-Trigger!', level)

    # No caso só queremos o alerta, mas temos que retornar um corpo vazio para
    # ser uma função de rota válida. Agora 'flash' pode ser chamado em qualquer
    # rota e um alerta sempre vai ser criado com a mensagem.
    return ''
```

## Evitando problemas futuros

A funcionalidade está quase pronta, mas precisamos melhorar o nosso _middleware_. O que acontece se durante uma
outra função de rota adicionarmos um outro trigger à resposta? O _middleware_ vai sobrescrever o _header_ `HX-Request` e
um _bug_ vai surgir na sua aplicação.

O _header_ `HX-Trigger` pode receber uma _string_ (que pode ser dividida por vírgulas) ou um objeto. Vamos modificar
nossa função `flash_to_htmx_trigger` para levar isso em conta. Em `app.py`:

```python
@app.after_request
def flash_to_htmx_trigger(response):
    """Middleware que converte flashes para HX-Trigger."""
    flashes = session.pop('_flashes') if '_flashes' in session else []

    if not flashes:
      return response

    trigger_content = {'alerts': flashes}

    if hx_header := response.headers.get('HX-Trigger'):
        try:
            existing_trigger_content = json.loads(hx_header)
        except json.JSONDecodeError:
            # É uma string, criar objeto vazio tendo ela como chave.
            for key in hx_header.split(','):
                trigger_content[key.strip()] = {}
        else:
            # É um objeto, fazer o merge com o nosso.
            if isinstance(existing_trigger_content, list):
                raise TypeError('HX-Trigger does not support array')

            trigger_content |= existing_trigger_content

    response.headers['HX-Trigger'] = json.dumps(trigger_content)

    return response
```

Com isso temos nosso alertas dinâmicos! Ainda há espaço para melhorias:
- O que acontece se tivermos muitos alertas?
- Como adicionar uma animação na remoção do alerta?
- Como tratar requisições falhas com HTMX que deveriam retornar uma resposta com o trigger?

E claro, testes unitários!

Os códigos podem ser acessados no repositório no [Github](https://github.com/gabrielmotaa/flask-htmx-alert).
