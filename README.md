# Sistemas Embarcados e IoT — Disciplina Prof. Navarro

# SISTEMAS EMBARCADOS E IoT

## Protocolo MQTT: Retain Flag e Last Will Testament

---

## Relatório Técnico — Trabalho Avaliativo

<br><br><br>

**Breno Henrique   |   Caio Henrique**

**Professor: Fabio Navarro**

**2026**

---

## 1. Introdução

O protocolo MQTT (Message Queuing Telemetry Transport) é amplamente utilizado em aplicações de Internet das Coisas (IoT) por ser leve, eficiente e projetado para redes instáveis com dispositivos de baixo consumo energético. Entre os recursos que tornam o MQTT confiável em ambientes reais, dois se destacam pela sua importância prática: a Retain Flag e o Last Will and Testament (LWT).

Ambos os mecanismos resolvem um problema fundamental do IoT: a comunicação entre dispositivos que frequentemente ficam offline, se reconectam e precisam garantir que informações críticas sejam preservadas e entregues corretamente. Este relatório descreve em detalhes o que são, como funcionam, quando usar cada um, as diferenças entre eles e como implementá-los em código real.

---

## 2. Retain Flag

### 2.1  O que é

A Retain Flag é um recurso do protocolo MQTT que instrui o broker a armazenar a última mensagem publicada em um determinado tópico. Quando um novo subscriber se conecta e assina esse tópico, ele recebe imediatamente essa mensagem retida, sem precisar esperar a próxima publicação.

Em termos simples: imagine um quadro de avisos na entrada de um prédio. Qualquer pessoa que entre, independentemente do horário, vê o último aviso afixado. A Retain Flag funciona da mesma forma: o broker guarda a mensagem como se fosse esse aviso, e quem chegar depois ainda a recebe.

### 2.2  Como funciona internamente

Quando um publisher envia uma mensagem com a retain flag ativada (retain = true), o broker armazena essa mensagem associada ao tópico. Apenas a última mensagem com retain é guardada por tópico. A mensagem é entregue imediatamente a todos os subscribers atualmente conectados, e também enviada automaticamente a qualquer novo subscriber que assinar aquele tópico no futuro.

Para limpar uma mensagem retida de um tópico, publica-se uma mensagem vazia (payload vazio) com retain = true no mesmo tópico. O broker então descarta a mensagem armazenada.

### 2.3  Por que usar

Em cenários IoT, sensores publicam dados periodicamente — a cada 30 segundos, um minuto, ou até mais tempo. Se um dashboard conectar ao broker entre duas publicações, sem a Retain Flag ele ficaria com a tela em branco até o próximo envio do sensor. Com a Retain Flag, o dashboard já recebe o valor mais recente ao se conectar, proporcionando uma experiência imediata e correta.

Outros cenários onde a Retain Flag é essencial:

Status de dispositivos: um sensor publicar seu status (online/offline) com retain garante que qualquer monitor que conecte saiba imediatamente o estado atual.

Configurações iniciais: um servidor de configuração publica parâmetros com retain, e os dispositivos os recebem assim que conectam, sem depender de novo envio.

Valores de referência: limites, metas ou setpoints que raramente mudam mas precisam estar disponíveis imediatamente a novos subscribers.

### 2.4  Casos de uso em código

O código a seguir demonstra um publisher Node.js enviando dados de temperatura com Retain Flag ativada, e um subscriber que, ao se conectar, recebe imediatamente o último valor publicado.

```javascript
// Publisher com Retain Flag — Node.js

// ── retain_publisher.js ─────────────────────────────────────

const mqtt = require('mqtt');

// conecta no broker HiveMQ Cloud (porta 8883 = MQTT com TLS)
const client = mqtt.connect('mqtts://broker.hivemq.com:8883', {
  clientId: 'pub-estufa-001',
  username: 'aluno',
  password: 'Aluno123'
});

client.on('connect', () => {
  console.log('Publisher conectado ao broker.');

  // simula leitura de sensor a cada 10 segundos
  setInterval(() => {
    const temperatura = (22 + Math.random() * 8).toFixed(1);

    const payload = JSON.stringify({
      sensor: 'temp-estufa-01',
      valor: parseFloat(temperatura),
      unidade: 'celsius',
      timestamp: new Date().toISOString()
    });

    // terceiro argumento: { retain: true } ativa a Retain Flag
    // o broker armazenará essa mensagem para novos subscribers
    client.publish('estufa/temperatura', payload, { retain: true }, (err) => {
      if (!err) console.log('Publicado com retain:', payload);
    });

  }, 10000);

});
```

```javascript
// Subscriber com Retain Flag — Node.js

// ── retain_subscriber.js ────────────────────────────────────

const mqtt = require('mqtt');

const client = mqtt.connect('mqtts://broker.hivemq.com:8883', {
  clientId: 'sub-dashboard-001',
  username: 'aluno',
  password: 'Aluno123'
});

client.on('connect', () => {
  console.log('Dashboard conectado.');

  // ao assinar, o broker já entrega a última mensagem retida
  // sem precisar esperar o próximo publish do sensor
  client.subscribe('estufa/temperatura', { qos: 1 });
  console.log('Aguardando dados (valor retido chega imediatamente)...');
});

client.on('message', (topic, payload) => {
  const dado = JSON.parse(payload.toString());

  console.log('Temperatura recebida:', dado.valor, dado.unidade);
  console.log('Publicado em:', dado.timestamp);

  // se for o valor retido, aparece IMEDIATAMENTE ao conectar
  // sem a Retain Flag, o dashboard ficaria em branco até o próximo envio
});

// Limpar mensagem retida: publicar payload vazio com retain: true
// client.publish('estufa/temperatura', '', { retain: true });
```

```cpp
// ESP32 — Retain Flag em C++

// ── ESP32 com Retain Flag (Arduino/C++) ─────────────────────

#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>

WiFiClientSecure espClient;
PubSubClient client(espClient);

void callback(char* topic, byte* payload, unsigned int length) {
  String mensagem = "";

  for (int i = 0; i < length; i++) {
    mensagem += (char)payload[i];  // converte bytes para String
  }

  Serial.print("Tópico: "); Serial.println(topic);
  Serial.print("Mensagem: "); Serial.println(mensagem);

  // Se chegou imediatamente ao conectar = era mensagem retida
}

void publicarComRetain(float temperatura) {
  String payload = "{\"temp\":";
  payload += String(temperatura, 1);
  payload += "}";

  // publish(topic, payload, retain)
  // true = ativa retain flag
  client.publish("estufa/temperatura", payload.c_str(), true);
}
```

---

## 3. Last Will and Testament (LWT)

### 3.1  O que é

O Last Will and Testament (LWT), ou "Última Vontade e Testamento" em tradução livre, é um mecanismo do MQTT pelo qual o cliente registra, no momento da conexão, uma mensagem que o broker publicará automaticamente caso aquele cliente se desconecte de forma inesperada.

É a solução do MQTT para um dos maiores desafios do IoT: detectar quando um dispositivo ficou offline sem avisar. Em redes reais, sensores podem perder energia, travar, ter a conexão WiFi interrompida, ou simplesmente parar de funcionar — e nesses casos não há como o dispositivo enviar um aviso de saída.

### 3.2  Como funciona internamente

O mecanismo opera em três fases distintas:

Registro (no momento da conexão): o cliente informa ao broker o tópico, o payload, o QoS e se a mensagem deve ser retida. O broker armazena essas informações internamente.

Monitoramento (durante a sessão): o broker aguarda as mensagens de keep-alive (PINGREQ/PINGRESP) do cliente. Se não receber essas mensagens dentro do intervalo configurado, considera o cliente desconectado.

Publicação automática (na desconexão inesperada): o broker publica a mensagem LWT no tópico registrado, notificando todos os subscribers daquele tópico sobre a queda do dispositivo.

Importante: se o cliente se desconectar de forma limpa (enviando o pacote DISCONNECT), o broker descarta a mensagem LWT e não a publica. O LWT só é acionado em desconexões não planejadas.

### 3.3  Por que usar

O LWT é essencial em qualquer sistema IoT onde saber que um dispositivo ficou offline é tão importante quanto os dados que ele envia. Exemplos práticos:

Monitoramento de saúde de frota: caminhões com sensores GPS que param de responder podem indicar acidente ou falha mecânica — o sistema precisa saber imediatamente.

Automação residencial: se o controlador de um sistema de aquecimento cair, o servidor precisa saber para acionar um modo de segurança.

Estufa agrícola: se o ESP32 responsável por um sensor de incêndio cair, o sistema de alarme precisa ser notificado imediatamente, pois a proteção está comprometida.

Dashboards de monitoramento: exibir o status real de cada dispositivo (online/offline) em tempo real.

### 3.4  Casos de uso em código

```javascript
// LWT — Dispositivo (Node.js)

// ── lwt_dispositivo.js ───────────────────────────────────────

const mqtt = require('mqtt');

const client = mqtt.connect('mqtts://broker.hivemq.com:8883', {
  clientId: 'esp32-sensor-incendio',
  username: 'aluno',
  password: 'Aluno123',

  // ── configuração do LWT ──────────────────────────────────
  will: {
    topic:   'estufa/dispositivos/incendio/status',
    payload: JSON.stringify({
      status:    'offline',
      motivo:    'desconexao_inesperada',
      timestamp: null
    }),
    qos:    1,
    retain: true
  }
});

client.on('connect', () => {
  console.log('Sensor de incêndio conectado.');

  client.publish(
    'estufa/dispositivos/incendio/status',
    JSON.stringify({ status: 'online', timestamp: new Date().toISOString() }),
    { retain: true, qos: 1 }
  );

  setInterval(() => {
    const dado = { fumaca: false, timestamp: new Date().toISOString() };
    client.publish('estufa/alerta/incendio', JSON.stringify(dado), { qos: 2 });
  }, 5000);
});
```

```javascript
// LWT — Monitor Central (Node.js)

// ── lwt_monitor.js ──────────────────────────────────────────

const mqtt = require('mqtt');

const monitor = mqtt.connect('mqtts://broker.hivemq.com:8883', {
  clientId: 'monitor-central',
  username: 'aluno',
  password: 'Aluno123'
});

monitor.on('connect', () => {
  monitor.subscribe('estufa/dispositivos/incendio/status', { qos: 1 });
  monitor.subscribe('estufa/dispositivos/+/status');
  console.log('Monitor central ativo.');
});

monitor.on('message', (topic, payload) => {
  const dado = JSON.parse(payload.toString());

  if (dado.status === 'offline') {
    console.log('ALERTA: dispositivo ficou offline!');
    console.log('Tópico:', topic);
    console.log('Motivo:', dado.motivo);
    acionarProtocoloDeEmergencia(topic);
  } else {
    console.log('Dispositivo online:', topic);
  }
});

function acionarProtocoloDeEmergencia(topico) {
  console.log('[EMERGÊNCIA] Protocolo acionado para:', topico);
}
```

```cpp
// ESP32 — LWT em C++

// ── ESP32 com LWT (Arduino/C++) ──────────────────────────────

#include <WiFi.h>
#include <WiFiClientSecure.h>
#include <PubSubClient.h>

WiFiClientSecure espClient;
PubSubClient client(espClient);

const char* TOPICO_STATUS = "estufa/dispositivos/incendio/status";

void reconnect() {
  while (!client.connected()) {
    Serial.print("Conectando ao broker...");

    if (client.connect(
          "esp32-incendio-01",
          "aluno",
          "Aluno123",
          TOPICO_STATUS,
          1,
          true,
          "{\"status\":\"offline\"}"
        )) {

      Serial.println(" conectado!");

      client.publish(TOPICO_STATUS,
                     "{\"status\":\"online\"}",
                     true);

      client.subscribe("estufa/alerta/#");

    } else {
      Serial.print(" falhou. rc=");
      Serial.print(client.state());
      Serial.println(" tentando novamente em 3s");
      delay(3000);
    }
  }
}

void loop() {
  if (!client.connected()) reconnect();
  client.loop();
}
```

---

## 4. Diferenças entre Retain Flag e LWT

Aspecto
Retain Flag
Last Will and Testament

Quem configura
O publisher, no momento de cada publish
O cliente, no momento da conexão (connect)

Quando é publicado
Imediatamente, junto com o publish normal
Somente quando o cliente se desconectar inesperadamente

Quem publica
O próprio publisher (com flag ativada)
O broker (automaticamente, sem ação do cliente)

Propósito principal
Garantir que novos subscribers recebam o último valor conhecido
Notificar outros dispositivos sobre uma desconexão não planejada

Gatilho
Novo subscriber assina o tópico
Timeout de keep-alive ou queda de rede

Conteúdo da mensagem
Dados operacionais (temperatura, nível, status)
Aviso de falha (status offline, motivo)

Uso típico
Sensores com publicação periódica, configurações, setpoints
Monitoramento de disponibilidade, alertas de falha

Analogia
Quadro de avisos com o último comunicado afixado
Testamento: instrução deixada para caso algo de errado aconteça

---

## 5. Uso combinado em um sistema real

```javascript
// Retain Flag + LWT combinados — Node.js

// ── sistema_completo.js ─────────────────────────────────────

const mqtt = require('mqtt');

const dispositivo = mqtt.connect('mqtts://broker.hivemq.com:8883', {
  clientId:  'esp32-nivel-agua',
  username:  'aluno',
  password:  'Aluno123',

  will: {
    topic:   'estufa/agua/status',
    payload: JSON.stringify({ status: 'offline', alerta: true }),
    qos:     1,
    retain:  true
  }
});

dispositivo.on('connect', () => {
  dispositivo.publish('estufa/agua/status',
    JSON.stringify({ status: 'online', timestamp: new Date().toISOString() }),
    { retain: true, qos: 1 }
  );

  setInterval(() => {
    const nivel = Math.floor(40 + Math.random() * 60);

    dispositivo.publish('estufa/agua/nivel',
      JSON.stringify({ nivel, unidade: '%', timestamp: new Date().toISOString() }),
      { retain: true, qos: 1 }
    );

    console.log('Nível publicado:', nivel + '%');

  }, 30000);

});

const dashboard = mqtt.connect('mqtts://broker.hivemq.com:8883', {
  clientId: 'dashboard-web'
});

setTimeout(() => {
  dashboard.on('connect', () => {
    dashboard.subscribe('estufa/agua/status');
    dashboard.subscribe('estufa/agua/nivel');
  });

  dashboard.on('message', (topic, payload) => {
    const d = JSON.parse(payload.toString());

    if (topic.includes('status')) {
      console.log('[Dashboard] Status do sensor:', d.status);
    } else {
      console.log('[Dashboard] Nível da água:', d.nivel + d.unidade);
    }
  });

}, 5000);
```

---

## 6. Resumo e boas práticas

### 6.1  Quando usar cada recurso

Cenário
Usar Retain Flag?
Usar LWT?

Sensor publica temperatura periodicamente
Sim — dashboard recebe valor imediato ao conectar
Opcional — se monitorar disponibilidade importar

Detector de incêndio
Sim — manter último estado de alerta retido
Sim — crítico saber se o sensor caiu

Configuração de dispositivos
Sim — dispositivos recebem configs ao conectar
Não

Monitoramento de frota de veículos
Sim — último status de cada veículo
Sim — detectar veículo sem sinal

Dados em alta frequência (temperatura a cada 1s)
Sim — mas atenção: retain guarda só a última mensagem
Opcional

### 6.2  Boas práticas

Combine LWT com retain: configure o LWT com retain = true para que o status de 'offline' fique disponível para futuros subscribers também.

Sempre publique status 'online' com retain ao conectar: o LWT cuida do offline, mas você precisa publicar o online explicitamente ao iniciar a sessão.

Use QoS 1 no LWT: garante que o broker entregue o aviso de desconexão aos subscribers. QoS 0 pode perder o aviso justamente quando a rede está instável.

Limpe mensagens retidas obsoletas: quando um tópico não é mais usado, publique um payload vazio com retain = true para limpar o broker.

Não use retain em dados de alta frequência com subscribers lentos: o retain sempre entrega o último valor, então em publicações muito rápidas a mensagem retida pode ser uma versão antiga por microsegundos.

---

## 7. Conclusão

A Retain Flag e o Last Will and Testament são dois dos recursos mais importantes do protocolo MQTT para aplicações IoT robustas. Enquanto a Retain Flag resolve o problema da disponibilidade imediata de dados para novos subscribers, o LWT resolve o problema crítico da detecção de falhas e desconexões inesperadas.

Quando utilizados em conjunto, formam uma base sólida para sistemas IoT confiáveis: qualquer componente que se conecte ao broker, em qualquer momento, terá acesso ao último estado conhecido de cada dispositivo e será automaticamente notificado caso algum dispositivo saia do ar sem aviso prévio.

Compreender esses mecanismos vai além de decorar parâmetros de funções — representa entender como sistemas distribuídos e assíncronos podem ser projetados para lidar com a realidade de redes instáveis e dispositivos com recursos limitados, que são as condições exatas sob as quais o IoT opera no mundo real.

---

**Breno Henrique & Caio Henrique  |  Sistemas Embarcados e IoT  |  Prof. Fabio Navarro  |  2026**

**Breno Henrique | Caio Henrique**
