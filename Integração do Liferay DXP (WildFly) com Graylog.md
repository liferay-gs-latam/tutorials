# Integração do Liferay DXP (WildFly) com Graylog

**Documento de referência técnica**
Versão 1.0 — Julho/2026

---

## 1. Objetivo

Este documento descreve como centralizar os logs de uma instalação **Liferay DXP** executando sobre **WildFly** no **Graylog**, utilizando o protocolo **GELF** (Graylog Extended Log Format).

São apresentadas três abordagens, com prós e contras de cada uma, para que o time possa escolher a mais adequada ao ambiente:

| Opção | Descrição | Indicada quando... |
|---|---|---|
| **A** | Appender GELF no Log4j do Liferay | Você quer controlar os logs pelo painel do próprio DXP e enviar apenas os logs da aplicação |
| **B** | Handler GELF no subsistema de logging do WildFly | Você quer centralizar tudo (WildFly + Liferay) sem alterar o WAR |
| **C** | Coleta externa com Filebeat/Fluent Bit | Você prioriza resiliência e não quer tocar em nada do servidor de aplicação |

> **Importante — escopo de suporte:** a Liferay não oferece integração *out-of-the-box* com o Graylog. Esta configuração é considerada uma **customização** e sua manutenção fica fora do escopo do suporte oficial da Liferay. Recomenda-se documentar e versionar todos os arquivos alterados.

---

## 2. Pré-requisitos

- Liferay DXP 7.3 ou 7.4 (Log4j 2) implantado como `ROOT.war` no WildFly
- WildFly com acesso de rede ao servidor Graylog (porta padrão GELF: **12201**)
- Graylog instalado e acessível pela interface web
- Biblioteca **logstash-gelf** (`biz.paluch.logging:logstash-gelf`), versão 1.15.x ou superior
- Permissão para reiniciar o WildFly em janela de manutenção

> **Nota sobre versões:** se o ambiente for DXP 7.0/7.1 (Log4j 1.x), a sintaxe dos arquivos de configuração é diferente da apresentada aqui. Consulte o time técnico antes de aplicar.

---

## 3. Configuração do input no Graylog

Antes de configurar o WildFly/Liferay, prepare o Graylog para receber as mensagens:

1. Acesse a interface web do Graylog.
2. Navegue até **System → Inputs**.
3. Selecione o tipo **GELF UDP** (ou **GELF TCP**, se preferir garantia de entrega) e clique em **Launch new input**.
4. Configure:
   - **Node:** selecione o nó (ou marque *Global*)
   - **Bind address:** `0.0.0.0`
   - **Port:** `12201`
5. Clique em **Save** e verifique se o input aparece com status **Running**.

> **UDP vs TCP:** UDP tem menor overhead e não bloqueia a aplicação se o Graylog ficar indisponível, mas pode perder mensagens. TCP garante entrega, porém pode gerar backpressure na aplicação em caso de indisponibilidade do Graylog. Para logs de aplicação, UDP costuma ser suficiente; para logs de auditoria, prefira TCP ou a Opção C.

---

## 4. Opção A — Appender GELF no Log4j do Liferay

Nesta abordagem, o próprio Log4j do Liferay envia os logs diretamente ao Graylog. Os níveis de log continuam sendo gerenciados normalmente pelo painel **Control Panel → Server Administration → Log Levels**.

### 4.1. Instalar a biblioteca GELF no WAR

Copie o JAR da biblioteca para dentro do deployment do Liferay:

```
wildfly/standalone/deployments/ROOT.war/WEB-INF/lib/logstash-gelf-1.15.1.jar
```

> **Alternativa mais "limpa":** criar um módulo WildFly (ver seção 5.1) e referenciá-lo no `jboss-deployment-structure.xml` do `ROOT.war`. A cópia direta em `WEB-INF/lib`, porém, é mais simples e funciona.

### 4.2. Criar o arquivo de extensão do Log4j

Crie o arquivo:

```
wildfly/standalone/deployments/ROOT.war/WEB-INF/classes/META-INF/portal-log4j-ext.xml
```

Com o conteúdo:

```xml
<?xml version="1.0"?>
<Configuration strict="true">
    <Appenders>
        <Gelf name="GELF"
              host="udp:SEU_HOST_GRAYLOG"
              port="12201"
              version="1.1"
              extractStackTrace="true"
              filterStackTrace="true"
              originHost="%host{fqdn}"
              includeFullMdc="true">
            <Field name="environment" literal="producao"/>
            <Field name="application" literal="liferay-dxp"/>
        </Gelf>
    </Appenders>

    <Loggers>
        <Root level="INFO">
            <AppenderRef ref="GELF"/>
        </Root>
    </Loggers>
</Configuration>
```

Ajustes possíveis:

- `host`: use `udp:` ou `tcp:` conforme o input criado no Graylog.
- `Field`: campos adicionais que aparecerão como atributos pesquisáveis no Graylog (ambiente, aplicação, datacenter etc.). Adicione quantos precisar.
- `level` do Root logger: controla o nível mínimo enviado ao Graylog.

> **Como funciona:** durante o startup, o Liferay carrega o `portal-log4j.xml` interno e, em seguida, aplica as extensões definidas em `portal-log4j-ext.xml`. Este é o mecanismo padrão de extensão de logging do produto (válido do 7.0 GA1 em diante), portanto o arquivo sobrevive a upgrades de fixpack — mas **deve ser reaplicado** se o WAR for substituído em um upgrade de versão.

### 4.3. Reiniciar o WildFly

```bash
./standalone.sh   # ou reinício via serviço (systemctl restart wildfly)
```

---

## 5. Opção B — Handler GELF no subsistema de logging do WildFly

Nesta abordagem, o **WildFly** envia ao Graylog tudo o que passa pelo seu subsistema de logging: logs do próprio servidor (deploy, datasources, subsistemas) **e** o que o Liferay imprime no console. Não é necessário alterar o WAR.

### 5.1. Criar o módulo WildFly

Crie a estrutura de diretórios:

```
wildfly/modules/biz/paluch/logging/main/
├── logstash-gelf-1.15.1.jar
└── module.xml
```

Conteúdo do `module.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<module xmlns="urn:jboss:module:1.1" name="biz.paluch.logging">
    <resources>
        <resource-root path="logstash-gelf-1.15.1.jar"/>
    </resources>
    <dependencies>
        <module name="org.jboss.logging"/>
    </dependencies>
</module>
```

### 5.2. Configurar o custom-handler

Edite o `wildfly/standalone/configuration/standalone.xml` e, dentro do subsistema `urn:jboss:domain:logging`, adicione:

```xml
<custom-handler name="GELF"
                class="biz.paluch.logging.gelf.wildfly.WildFlyGelfLogHandler"
                module="biz.paluch.logging">
    <level name="INFO"/>
    <properties>
        <property name="host" value="udp:SEU_HOST_GRAYLOG"/>
        <property name="port" value="12201"/>
        <property name="version" value="1.1"/>
        <property name="extractStackTrace" value="true"/>
        <property name="filterStackTrace" value="true"/>
        <property name="includeFullMdc" value="true"/>
        <property name="additionalFields" value="environment=producao,application=liferay-dxp"/>
    </properties>
</custom-handler>
```

E inclua o handler no root logger:

```xml
<root-logger>
    <level name="INFO"/>
    <handlers>
        <handler name="CONSOLE"/>
        <handler name="FILE"/>
        <handler name="GELF"/>
    </handlers>
</root-logger>
```

### 5.3. Alternativa: configuração via CLI (sem parar o servidor)

```bash
./jboss-cli.sh --connect

/subsystem=logging/custom-handler=GELF:add( \
    class="biz.paluch.logging.gelf.wildfly.WildFlyGelfLogHandler", \
    module="biz.paluch.logging", \
    level=INFO, \
    properties={ \
        host="udp:SEU_HOST_GRAYLOG", \
        port="12201", \
        version="1.1", \
        extractStackTrace="true", \
        filterStackTrace="true", \
        includeFullMdc="true", \
        additionalFields="environment=producao,application=liferay-dxp" \
    })

/subsystem=logging/root-logger=ROOT:add-handler(name=GELF)
```

### 5.4. Reiniciar (se editou o XML manualmente)

```bash
systemctl restart wildfly   # ou o método usado no ambiente
```

> **Atenção:** não habilite as Opções A e B simultaneamente se o Liferay também estiver logando no console — isso gera **mensagens duplicadas** no Graylog.

---

## 6. Opção C — Coleta externa com Filebeat ou Fluent Bit (recomendada para produção)

Nesta abordagem, o Liferay/WildFly continuam gravando logs em arquivo normalmente, e um agente coletor externo lê os arquivos e envia ao Graylog. É o desenho mais resiliente:

- **Nenhuma alteração** no WildFly ou no Liferay
- Se o Graylog cair, **nenhum log é perdido** (o agente retoma a leitura do arquivo)
- Sem risco de backpressure na aplicação
- Padrão amplamente usado em ambientes containerizados (sidecar em Kubernetes/Docker)

### 6.1. Arquivos de log a coletar

| Arquivo | Conteúdo |
|---|---|
| `wildfly/standalone/log/server.log` | Logs do WildFly + console do Liferay |
| `liferay-home/logs/liferay.YYYY-MM-DD.log` | Logs do Liferay em arquivo próprio |

### 6.2. Exemplo com Fluent Bit (output GELF nativo)

Arquivo `fluent-bit.conf`:

```ini
[SERVICE]
    Flush         5
    Log_Level     info

[INPUT]
    Name          tail
    Path          /opt/wildfly/standalone/log/server.log
    Tag           liferay.server
    Multiline.parser  java

[OUTPUT]
    Name                    gelf
    Match                   liferay.*
    Host                    SEU_HOST_GRAYLOG
    Port                    12201
    Mode                    udp
    Gelf_Short_Message_Key  log
```

> O parser `multiline java` é importante para que stack traces cheguem ao Graylog como **uma única mensagem**, e não uma linha por vez.

### 6.3. Exemplo com Filebeat (via Graylog Sidecar ou standalone)

Arquivo `filebeat.yml`:

```yaml
filebeat.inputs:
  - type: filestream
    id: liferay-server-log
    paths:
      - /opt/wildfly/standalone/log/server.log
    parsers:
      - multiline:
          type: pattern
          pattern: '^\d{4}-\d{2}-\d{2}'
          negate: true
          match: after

output.logstash:
  hosts: ["SEU_HOST_GRAYLOG:5044"]
```

> Para Filebeat, crie no Graylog um input do tipo **Beats** (porta 5044) em vez de GELF. Se a empresa já utiliza o **Graylog Sidecar**, a configuração do Filebeat pode ser gerenciada centralmente pela interface do Graylog.

---

## 7. Validação

Após aplicar a opção escolhida:

1. Reinicie o serviço (WildFly ou o agente coletor).
2. No Graylog, acesse **System → Inputs** e verifique o contador de **Throughput** do input — ele deve começar a subir.
3. Acesse a tela **Search** e execute uma busca sem filtro nos últimos 5 minutos.
4. Confirme que os campos adicionais (`environment`, `application`) aparecem nas mensagens.
5. Force um log de teste: acesse **Control Panel → Server Administration → Log Levels** no Liferay, ajuste uma categoria para `DEBUG` temporariamente e verifique a chegada das mensagens.
6. Provoque uma exceção controlada (ex.: URL inválida de um portlet) e confirme que o **stack trace chega como mensagem única**.

---

## 8. Troubleshooting

| Sintoma | Causa provável | Ação |
|---|---|---|
| Nenhuma mensagem chega ao Graylog | Firewall bloqueando a porta 12201 | Testar com `nc -u SEU_HOST_GRAYLOG 12201` a partir do servidor WildFly |
| Erro `ClassNotFoundException: ...GelfLogHandler` no boot | JAR não encontrado no módulo/WAR | Conferir caminho do JAR e o `module.xml` |
| Mensagens truncadas via UDP | Payload maior que o MTU da rede | Usar GELF TCP, ou habilitar chunking (padrão do GELF UDP) |
| Stack traces quebrados em várias mensagens | Falta de parser multiline (Opção C) | Configurar `Multiline.parser java` (Fluent Bit) ou `multiline` (Filebeat) |
| Mensagens duplicadas | Opções A e B ativas ao mesmo tempo | Manter apenas uma das opções |
| Configuração perdida após upgrade do DXP | WAR substituído no upgrade | Reaplicar o `portal-log4j-ext.xml` e o JAR no novo WAR (Opção A) |

---

## 9. Recomendações finais

- **Para produção**, prefira a **Opção C** (coleta externa): é a mais resiliente e a única que não exige alterações no servidor de aplicação.
- Se optar pelas Opções A ou B, **versionar** os arquivos alterados (`portal-log4j-ext.xml`, `standalone.xml`, `module.xml`) no repositório de configuração do ambiente.
- Definir **streams e alertas** no Graylog para os níveis `ERROR`/`FATAL` da aplicação.
- Configurar **retenção de índices** no Graylog conforme a política de dados da empresa.
- Testar a integração primeiro em ambiente de **homologação**, incluindo o comportamento durante indisponibilidade do Graylog.
- Lembrar que esta integração é uma **customização não coberta pelo suporte oficial da Liferay**; incidentes relacionados exclusivamente ao envio de logs devem ser tratados pelo time interno.

---

## 10. Referências

- Documentação do GELF: https://go2docs.graylog.org/current/getting_in_log_data/gelf.html
- Biblioteca logstash-gelf: https://logging.paluch.biz/
- Liferay — Using a Custom Service (DXP Cloud): https://learn.liferay.com/w/dxp/cloud
- Fluent Bit — GELF output: https://docs.fluentbit.io/manual/pipeline/outputs/gelf
- Graylog Sidecar: https://go2docs.graylog.org/current/getting_in_log_data/graylog_sidecar.html
