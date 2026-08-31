# workshop-demo-log-pipeline-config

Configuración de despliegue del pipeline de logs de auditoría. Es lo que
sincroniza Argo CD; el código vive en
[`workshop-demo-log-pipeline`](https://github.com/rh-workshop/workshop-demo-log-pipeline).

## Qué se despliega

| Recurso | Para qué |
|---|---|
| 4 `KafkaTopic` | Cifrado y enmascarado, cada uno con su cola de descarte |
| 3 `KafkaUser` | Una identidad por servicio, con permisos de mínimo privilegio |
| `log-processor` | Descifra y enmascara los datos personales |
| `log-sink` | Entrega al destino final; **no monta la llave de cifrado** |
| `audit-producer` | Emisor de referencia en Java — solo en dev y test |
| `dotnet-log-producer` | Host que demuestra el paquete .NET — solo en dev y test |

## Diferencias por ambiente

| Ambiente | Servicios |
|---|---|
| dev, test | Los cuatro: processor, sink y las dos piezas de demostración |
| prod, contingencia | Solo processor y sink |

En producción quien publica son los microservicios que integran el paquete .NET
([`workshop-demo-log-producer`](https://github.com/rh-workshop/workshop-demo-log-producer)).
Las dos piezas de demostración —el emisor Java y el host .NET— inyectarían
eventos ficticios en el tópico de auditoría, así que los overlays las retiran.

El host .NET vive aquí y no en un repositorio de configuración propio porque lo
que se entrega a los equipos es el **paquete**, no un servicio: el host solo
demuestra la librería dentro de este mismo flujo.

## Requisitos previos

- El clúster Kafka de la plataforma, que despliega
  [`workshop-demo-platform-config`](https://github.com/rh-workshop/workshop-demo-platform-config).
- Un Secret `kv-demo` con la llave de cifrado (clave `aes-key`), que montan el
  processor y el emisor de referencia. El sink no lo necesita.

## Promocionar una versión

El registro y el tag se fijan en el bloque `images:` del `base`: cambiar de
versión es una línea, y el tag es el commit que publicó el pipeline de CI.
