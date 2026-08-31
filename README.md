# workshop-demo-kafka-audit-pipeline-config

Configuración de despliegue del pipeline de logs de auditoría. Es lo que
sincroniza Argo CD; el código vive en
[`workshop-demo-kafka-audit-pipeline`](https://github.com/rh-workshop/workshop-demo-kafka-audit-pipeline).

## Qué se despliega

| Recurso | Para qué |
|---|---|
| 4 `KafkaTopic` | Cifrado y enmascarado, cada uno con su cola de descarte |
| 3 `KafkaUser` | Una identidad por servicio, con permisos de mínimo privilegio |
| `log-processor` | Descifra y enmascara los datos personales |
| `log-sink` | Entrega al destino final; **no monta la llave de cifrado** |
| `dummy-data-producer` | Emite eventos de prueba con **datos ficticios** (Java) — solo dev y test |
| `dummy-data-producer-dotnet` | Igual, con el paquete .NET — solo dev y test |

## Diferencias por ambiente

| Ambiente | Servicios |
|---|---|
| dev, test | Los cuatro: processor, sink y las dos piezas de datos ficticios |
| prod, contingencia | Solo processor y sink |

Cada servicio es una carpeta bajo `apps/`, con su `base/` y un `overlays/<ambiente>`
por cada ambiente donde existe:

```
apps/
├── log-processor/        base + overlays dev, test, prod, contingencia
├── log-sink/             base + overlays dev, test, prod, contingencia
└── dummy-data-producer/  base + overlays dev, test        (no existe en prod)
```

Es la misma convención que `workshop-demo-app-config`, y no es cosmética: los
ApplicationSets descubren servicios con el patrón `apps/*/overlays/<ambiente>`. Un
servicio entra en un ambiente creando su carpeta, y **la ausencia de carpeta es lo que
lo mantiene fuera** — por eso el emisor de datos ficticios no puede desplegarse en
producción aunque alguien lo intente: allí no hay overlay que lo describa.

Además usan **imágenes distintas**: producción despliega `kafka-audit-pipeline`, que
no contiene el código generador de datos ficticios —vive en otro artefacto—, así que
el aislamiento no depende de estos manifiestos.

En producción quien publica son los microservicios que integran el paquete .NET
([`workshop-demo-kafka-audit-producer`](https://github.com/rh-workshop/workshop-demo-kafka-audit-producer)).
Los emisores de prueba generan transacciones inventadas —correos, cédulas y
tarjetas ficticias—; en producción se mezclarían con las reales en el tópico de
auditoría y después nadie podría distinguir cuáles ocurrieron de verdad.

El host .NET vive aquí y no en un repositorio de configuración propio porque lo
que se entrega a los equipos es el **paquete**, no un servicio: el host solo
demuestra la librería dentro de este mismo flujo.

## Requisitos previos

- El clúster Kafka de la plataforma, que despliega
  [`workshop-demo-platform-config`](https://github.com/rh-workshop/workshop-demo-platform-config).
- Un Secret `kv-demo` en el namespace `kafka` con la llave de cifrado (clave
  `aes-key`), que montan el processor y el emisor de referencia. El sink no lo
  necesita.

  **No está en Git ni puede estarlo**: es la llave maestra de la que se derivan las
  claves de cifrado. Se crea fuera del flujo GitOps, una vez por ambiente y con una
  llave DISTINTA en cada uno:

  ```
  oc create secret generic kv-demo -n kafka \
    --from-literal=aes-key="$(openssl rand -base64 32)"
  ```

  En un despliegue productivo esto lo sustituye un gestor de secretos (External
  Secrets Operator contra un vault corporativo), que además permite rotar la llave: el
  formato del payload cifrado va versionado precisamente para admitir esa rotación.
  Los Deployments llevan `reloader.stakater.com/auto`, así que reinician solos cuando
  el Secret cambia.

## Promocionar una versión

La imagen se fija **por digest en el overlay de cada ambiente**, nunca por tag en el
`base`. Un digest identifica un contenido exacto e inmutable: es lo que permite afirmar
que lo que se validó en test es byte a byte lo que corre en producción, y lo que hace
verificable la firma de cosign. Un tag como `latest` puede apuntar mañana a otra imagen.

El flujo, en dos tramos:

1. **CI → dev.** Al hacer push, el pipeline construye, firma y publica en la
   organización `company-dev`, y escribe el digest en `overlays/dev`.
2. **dev → test → prod.** Los `PipelineRun` de `.tekton/promote-*` copian la imagen
   entre organizaciones de Quay sin reconstruirla, y actualizan el digest del overlay
   destino. Se disparan **solo por Pull Request** contra las ramas `test` y `prod`: a un
   ambiente superior no se llega con un push directo. La firma se verifica siempre, y
   hacia producción se exige además que el commit venga etiquetado.

El nombre lógico del bloque `images:` (`kafka-audit-pipeline`) debe coincidir con el
parámetro `image-name` del PipelineRun; si divergen, la promoción no encuentra qué
sustituir y el digest se pierde sin que falle nada.
