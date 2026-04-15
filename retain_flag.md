# Retain Flag (Resumo)

## O que é
A Retain Flag é um recurso do MQTT que instrui o broker a armazenar a última mensagem publicada em um tópico. Qualquer novo subscriber que assinar esse tópico receberá imediatamente essa mensagem armazenada.

## Como funciona (resumo)
- O publisher envia uma mensagem com `retain = true`.
- O broker guarda apenas a última mensagem retida por tópico.
- Quando um novo subscriber assina o tópico, o broker entrega a mensagem retida instantaneamente.
- Para limpar a mensagem retida publica-se um payload vazio com `retain = true`.

## Quando usar
- Informações que devem estar disponíveis imediatamente (status de dispositivo, configuração, setpoints).
- Dashboards que precisam mostrar o último valor ao conectar.
- Não é ideal para dados em altíssima frequência se a retenção de apenas um valor não fizer sentido.

## Boas práticas
- Use para status e configurações iniciais.
- Evite reter mensagens de alta frequência sem necessidade.
- Limpe mensagens obsoletas publicando payload vazio com `retain = true`.

## Exemplo (Node.js) - publicar com retain
```javascript
// publish(..., { retain: true })
client.publish('estufa/temperatura', JSON.stringify({ valor: 23.5, timestamp: new Date().toISOString() }), { retain: true });
```

## Limpar mensagem retida
```javascript
client.publish('estufa/temperatura', '', { retain: true });
```

## Exemplos práticos

### Node.js — Publisher (com retain)
```javascript
// retain_publisher.js
const mqtt = require('mqtt');
const client = mqtt.connect('mqtt://test.mosquitto.org');

client.on('connect', () => {
  const payload = JSON.stringify({ sensor: 'temp-01', valor: 26.3, ts: new Date().toISOString() });
  // Mensagem será retida pelo broker
  client.publish('exemplo/estufa/temperatura', payload, { retain: true, qos: 1 }, () => {
    console.log('Publicado com retain:', payload);
    // fecha após publicar (opcional)
    client.end();
  });
});
```

### Node.js — Subscriber (recebe mensagem retida ao conectar)
```javascript
// retain_subscriber.js
const mqtt = require('mqtt');
const client = mqtt.connect('mqtt://test.mosquitto.org');

client.on('connect', () => {
  client.subscribe('exemplo/estufa/temperatura', { qos: 1 }, () => {
    console.log('Assinado em exemplo/estufa/temperatura — aguardando (mensagem retida chega imediatamente)');
  });
});

client.on('message', (topic, payload) => {
  console.log('Recebido:', topic, payload.toString());
});
```

Execução prática: primeiro rode `retain_publisher.js` para publicar a mensagem retida. Em seguida rode `retain_subscriber.js` — ele receberá imediatamente a última mensagem retida.

### ESP32 (Arduino) — publicar com retain
```cpp
// Exemplo simplificado — use PubSubClient
#include <WiFi.h>
#include <PubSubClient.h>

WiFiClient net;
PubSubClient client(net);

void publicarRetain(float temp) {
  char payload[64];
  snprintf(payload, sizeof(payload), "{\"temp\":%.1f,\"ts\":\"%s\"}", temp, "2026-04-14T12:00:00Z");
  client.publish("exemplo/estufa/temperatura", payload, true); // retain = true
}

// Ao conectar, se já existir mensagem retida no tópico, o callback de subscribe receberá ela imediatamente.
```
