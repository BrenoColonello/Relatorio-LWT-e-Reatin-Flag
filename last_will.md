# Last Will and Testament (LWT) — Resumo

## O que é
O LWT é uma mensagem registrada pelo cliente no momento da conexão. Se o cliente se desconectar de forma inesperada, o broker publica automaticamente essa mensagem no tópico configurado, informando outros subscribers sobre a queda.

## Como funciona (resumo)
- O cliente passa os parâmetros do LWT (tópico, payload, QoS, retain) ao conectar.
- O broker monitora a sessão (keep-alive). Se houver falha sem um DISCONNECT limpo, o broker publica o LWT.
- Se o cliente desconectar limpo, o broker descarta o LWT.

## Quando usar
- Monitoramento de disponibilidade (detectar dispositivos offline).
- Cenários críticos (sensores de incêndio, controladores de equipamentos).
- Combine com `retain = true` no LWT se quiser que o estado "offline" permaneça disponível para novos subscribers.

## Boas práticas
- Defina o LWT com `qos = 1` para maior confiabilidade.
- Publique explicitamente um status `online` com `retain` ao conectar.
- Use mensagens claras (status, motivo, timestamp).

## Exemplos práticos

### Node.js — Dispositivo com LWT (simula desconexão inesperada)
```javascript
// lwt_device.js
const mqtt = require('mqtt');
const client = mqtt.connect('mqtt://test.mosquitto.org', {
  clientId: 'lwt-sensor-01',
  will: {
    topic: 'exemplo/estufa/sensor-01/status',
    payload: JSON.stringify({ status: 'offline', motivo: 'desconexao_inesperada', ts: new Date().toISOString() }),
    qos: 1,
    retain: true
  }
});

client.on('connect', () => {
  console.log('Conectado e LWT registrado');
  // publica online com retain
  client.publish('exemplo/estufa/sensor-01/status', JSON.stringify({ status: 'online', ts: new Date().toISOString() }), { retain: true, qos: 1 });
  // Simula envio de dados e depois mata o processo sem desconectar limpo
  setTimeout(() => {
    console.log('Simulando falha: encerrando processo sem client.end()');
    process.exit(1); // desconexão inesperada -> broker publica LWT
  }, 5000);
});
```

### Node.js — Monitor que recebe LWT
```javascript
// lwt_monitor.js
const mqtt = require('mqtt');
const monitor = mqtt.connect('mqtt://test.mosquitto.org');

monitor.on('connect', () => {
  monitor.subscribe('exemplo/estufa/+/status', { qos: 1 });
  console.log('Monitor ativo, aguardando status...');
});

monitor.on('message', (topic, payload) => {
  console.log('Recebido:', topic, payload.toString());
});
```

Testando: rode `lwt_monitor.js`, depois `lwt_device.js`. Quando o dispositivo fizer exit sem `client.end()`, o broker publicará a mensagem LWT no tópico configurado.

### ESP32 (Arduino) — configurar LWT via PubSubClient
```cpp
// Ao chamar client.connect(), a API PubSubClient fornece overload que aceita parâmetros de LWT
// client.connect(clientId, username, password, willTopic, willQos, willRetain, willMessage);

// Exemplo:
client.connect("esp32-lwt-01", "aluno", "Aluno123", "exemplo/estufa/sensor-01/status", 1, true, "{\"status\":\"offline\"}");

// Após conectar, publique status online com retain:
client.publish("exemplo/estufa/sensor-01/status", "{\"status\":\"online\"}", true);
```

