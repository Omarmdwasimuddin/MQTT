# MQTT (NestJS Microservices)

**Source:** https://docs.nestjs.com/microservices/mqtt

[MQTT](https://mqtt.org/) (Message Queuing Telemetry Transport) হলো একটা open source, lightweight messaging protocol, যেটা low latency এর জন্য optimize করা। এটা একটা **publish/subscribe** model ব্যবহার করে device connect করার একটা scalable আর cost-efficient উপায় দেয়। MQTT এর উপর তৈরি একটা communication system এ থাকে publishing server, একটা broker, এবং এক বা একাধিক client। এই protocol টা constrained device গুলোর জন্য, এবং low-bandwidth, high-latency, বা unreliable network এর জন্য design করা।

---

## 1. Installation

MQTT-based microservice বানানো শুরু করার আগে, প্রথমে প্রয়োজনীয় package টা install করো:

```bash
npm i --save mqtt
```

---

## 2. Overview

MQTT transporter ব্যবহার করার জন্য, `createMicroservice()` method এ নিচের options object টা পাস করো:

```typescript
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
  },
});
```

> **Hint:** `Transport` enum টা `@nestjs/microservices` package থেকে import করা হয়।

---

## 3. Options

`options` object টা বেছে নেওয়া transporter অনুযায়ী নির্দিষ্ট। **MQTT** transporter [MQTT.js client options](https://github.com/mqttjs/MQTT.js/#mqttclientstreambuilder-options) গুলো expose করে, সাথে `url`, `subscribeOptions` (দেখো [Quality of Service](#5-quality-of-service-qos)), আর `userProperties` (দেখো [record builders](#7-record-builders)) property গুলোও। Server এ, `maxConnectionAttempts` property দিয়ে initial connection establish করার attempt সংখ্যা limit করা যায় (default: `-1`, অর্থাৎ unlimited)। Server connect হয়ে যাওয়ার পর reconnection এর ক্ষেত্রে এই limit apply হয় না।

---

## 4. Client

অন্য microservice transporter এর মতোই, একটা MQTT `ClientProxy` instance তৈরি করার জন্য তোমার কাছে [বেশ কিছু option](https://docs.nestjs.com/microservices/basics#client) আছে।

একটা instance তৈরি করার একটা উপায় হলো `ClientsModule` ব্যবহার করা। এটা import করো এবং এর `register()` method ব্যবহার করে উপরে `createMicroservice()` method এ দেখানো একই property গুলো সহ একটা options object পাস করো, সাথে injection token হিসেবে ব্যবহারের জন্য একটা `name` property। `ClientsModule` সম্পর্কে আরো জানতে overview এর [client section](https://docs.nestjs.com/microservices/basics#client) দেখো।

```typescript
@Module({
  imports: [
    ClientsModule.register([
      {
        name: 'MATH_SERVICE',
        transport: Transport.MQTT,
        options: {
          url: 'mqtt://localhost:1883',
        },
      },
    ]),
  ],
  // ...
})
```

`ClientProxyFactory` অথবা `@Client()` decorator দিয়েও client তৈরি করা যায়। দুটোই overview এর [client section](https://docs.nestjs.com/microservices/basics#client) এ describe করা আছে।

---

## 5. Context

আরো complex scenario তে, incoming request সম্পর্কে অতিরিক্ত তথ্য দরকার হতে পারে। MQTT transporter এ, `MqttContext` object access করা যায়।

```typescript
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

> **Hint:** `@Payload()`, `@Ctx()`, আর `MqttContext` — এগুলো `@nestjs/microservices` package থেকে import করা হয়।

Original MQTT [packet](https://github.com/mqttjs/mqtt-packet) access করতে, `MqttContext` object এর `getPacket()` method ব্যবহার করো:

```typescript
@MessagePattern('notifications')
getNotifications(@Payload() data: number[], @Ctx() context: MqttContext) {
  console.log(context.getPacket());
}
```

---

## 6. Wildcards

একটা subscription একটা explicit topic কে target করতে পারে, অথবা wildcard ও include করতে পারে। দুই ধরনের wildcard পাওয়া যায়: `+` একটা single topic level match করে, আর `#` একাধিক topic level match করে।

```typescript
@MessagePattern('sensors/+/temperature/+')
getTemperature(@Ctx() context: MqttContext) {
  console.log(`Topic: ${context.getTopic()}`);
}
```

---

## 7. Quality of Service (QoS)

Default ভাবে, `@MessagePattern()` আর `@EventPattern()` decorator দিয়ে তৈরি subscription গুলো QoS 0 ব্যবহার করে। যদি বেশি QoS দরকার হয়, তাহলে connection establish করার সময় `subscribeOptions` block দিয়ে global ভাবে সেটা সেট করো:

```typescript
const app = await NestFactory.createMicroservice<MicroserviceOptions>(AppModule, {
  transport: Transport.MQTT,
  options: {
    url: 'mqtt://localhost:1883',
    subscribeOptions: {
      qos: 2,
    },
  },
});
```

### 7.1 Per-pattern QoS

`@MessagePattern()` বা `@EventPattern()` decorator এর দ্বিতীয় argument, অর্থাৎ extras object এ একটা `qos` property পাস করে, একটা নির্দিষ্ট pattern এর জন্য subscription QoS override করা যায়। যেসব pattern এ নিজস্ব `qos` নেই, সেগুলো global `subscribeOptions.qos` value ব্যবহার করে।

```typescript
@EventPattern('critical-events', { qos: 2 })
handleCriticalEvent(@Payload() data: any) {
  // এই subscription QoS 2 ব্যবহার করে
}

@EventPattern('metrics', { qos: 0 })
handleMetrics(@Payload() data: any) {
  // এই subscription QoS 0 ব্যবহার করে
}
```

---

## 8. Record Builder

Message option গুলো configure করতে (QoS level adjust করা, Retain বা DUP flag সেট করা, অথবা payload এ property add করা), `MqttRecordBuilder` class ব্যবহার করো। যেমন, নিচের record টা `setQoS()` method দিয়ে QoS `1` সেট করে, আর `setProperties()` method দিয়ে একটা user property add করে:

```typescript
const userProperties = { 'x-version': '1.0.0' };
const record = new MqttRecordBuilder(':cat:')
  .setProperties({ userProperties })
  .setQoS(1)
  .build();
client.send('replace-emoji', record).subscribe(...);
```

> **Hint:** `MqttRecordBuilder` class টা `@nestjs/microservices` package থেকে export করা হয়।

Server side এ, `MqttContext` এর মাধ্যমে এই option গুলো read করা যায়:

```typescript
@MessagePattern('replace-emoji')
replaceEmoji(@Payload() data: string, @Ctx() context: MqttContext): string {
  const { properties: { userProperties } } = context.getPacket();
  return userProperties['x-version'] === '1.0.0' ? '🐱' : '🐈';
}
```

একটা client এর পাঠানো সব request এর জন্য user property configure করতে, সেগুলো `ClientProxyFactory` তে options হিসেবে পাস করো:

```typescript
import { Module } from '@nestjs/common';
import { ClientProxyFactory, Transport } from '@nestjs/microservices';

@Module({
  providers: [
    {
      provide: 'API_v1',
      useFactory: () =>
        ClientProxyFactory.create({
          transport: Transport.MQTT,
          options: {
            url: 'mqtt://localhost:1883',
            userProperties: { 'x-version': '1.0.0' },
          },
        }),
    },
  ],
})
export class ApiModule {}
```

---

## 9. Instance Status Update

Connection এবং underlying driver instance এর state সম্পর্কে real-time update পেতে, `status` stream এ subscribe করো। এই stream বেছে নেওয়া driver অনুযায়ী নির্দিষ্ট status update দেয়। MQTT driver এর ক্ষেত্রে, `status` stream `connected`, `disconnected`, `reconnecting`, আর `closed` — এই চার ধরনের event emit করে।

```typescript
this.client.status.subscribe((status: MqttStatus) => {
  console.log(status);
});
```

> **Hint:** `MqttStatus` type টা `@nestjs/microservices` package থেকে import করা হয়।

একইভাবে, server এর status সম্পর্কে notification পেতে, server এর `status` stream এও subscribe করা যায়।

```typescript
const server = app.connectMicroservice<MicroserviceOptions>(...);
server.status.subscribe((status: MqttStatus) => {
  console.log(status);
});
```

---

## 10. MQTT Event Listen করা

কিছু ক্ষেত্রে, microservice এর emit করা internal event গুলো listen করার দরকার হতে পারে। যেমন, কোনো error ঘটলে অতিরিক্ত operation trigger করার জন্য `error` event এর জন্য listen করতে পারো। এর জন্য, নিচের মতো `on()` method ব্যবহার করো:

```typescript
this.client.on('error', (err) => {
  console.error(err);
});
```

একইভাবে, server এর internal event ও listen করা যায়:

```typescript
server.on<MqttEvents>('error', (err) => {
  console.error(err);
});
```

> **Hint:** `MqttEvents` type টা `@nestjs/microservices` package থেকে import করা হয়।

---

## 11. Underlying Driver Access

আরো advanced use case এ, underlying driver instance access করার দরকার হতে পারে — যেমন, manually connection close করা বা driver-specific method ব্যবহার করা। তবে, বেশিরভাগ ক্ষেত্রে driver directly access করার **দরকার হয় না**।

এর জন্য, `unwrap()` method ব্যবহার করো, যেটা underlying driver instance return করে। Generic type parameter দিয়ে বলা যায় তুমি কোন type এর driver instance আশা করছো।

```typescript
const mqttClient = this.client.unwrap<import('mqtt').MqttClient>();
```

একইভাবে, server এর underlying driver instance ও access করা যায়:

```typescript
const mqttClient = server.unwrap<import('mqtt').MqttClient>();
```
