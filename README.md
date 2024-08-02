# Problemas CloudStack

###### Problema 1: `IllegalArgumentException` em `UploadVolumeCmd` e connection timeout em `UriUtils.java`
Ao tentar carregar um volume, a aplicação lança uma exceção `IllegalArgumentException` indicando que não é possível alcançar a URL(http://company.com/tools-lient-teste.qcow2) fornecida para o volume dentro do tempo limite configurado.

###### Erro
```
2020-10-26 02:16:06,586 ERROR [c.c.a.ApiAsyncJobDispatcher] (API-Job-Executor-95:ctx-02c30d5b job-2018) (logid:cfd92e1e) Unexpected exception while executing org.apache.cloudstack.api.command.user.volume.UploadVolumeCmd
java.lang.IllegalArgumentException: Cannot reach URL: http://company.com/tools-lient-teste.qcow2 due to: The host did not accept the connection within timeout of 5000 ms
at com.cloud.utils.UriUtils.checkUrlExistence(UriUtils.java:480)
at com.cloud.storage.VolumeApiServiceImpl.validateVolume(VolumeApiServiceImpl.java:420)
at com.cloud.storage.VolumeApiServiceImpl.uploadVolume(VolumeApiServiceImpl.java:304)
at com.cloud.storage.VolumeApiServiceImpl.uploadVolume(VolumeApiServiceImpl.java:184)
...
```

Método impactado: `checkUrlExistence` em `UriUtils.java`

Problema: Falha ao conectar-se ao URL do volume dentro do tempo limite de 5000 ms.

Possíveis causas:
- URL errado ou inacessível.
- Servidor fora do ar ou inacessível da rede.
- Problemas de rede ou configuração de firewall.

Possiveis soluções:

- Verificar se o URL é válido e acessível.
- Aumentar o tempo limite de conexão configurado no `HttpClient` para aguardar respostas lentas.
- Verificar se há restrições de rede ou firewall que possam estar bloqueando o acesso ao URL.
- Verificar as configurações do `VolumeApiServiceImpl`.

######  [Código principal](https://github.com/apache/cloudstack/blob/4.13/utils/src/main/java/com/cloud/utils/UriUtils.java#L466)

```
public static void checkUrlExistence(String url) {
        if (url.toLowerCase().startsWith("http") || url.toLowerCase().startsWith("https")) {
            HttpClient httpClient = getHttpClient();
            HeadMethod httphead = new HeadMethod(url);
            try {
                if (httpClient.executeMethod(httphead) != HttpStatus.SC_OK) {
                    throw new IllegalArgumentException("Invalid URL: " + url);
                }
                if (url.endsWith("metalink") && !checkUrlExistenceMetalink(url)) {
                    throw new IllegalArgumentException("Invalid URLs defined on metalink: " + url);
                }
            } catch (HttpException hte) {
                throw new IllegalArgumentException("Cannot reach URL: " + url + " due to: " + hte.getMessage());
            } catch (IOException ioe) {
                throw new IllegalArgumentException("Cannot reach URL: " + url + " due to: " + ioe.getMessage());
            } finally {
                httphead.releaseConnection();
            }
        }
    }
...
```

######  Problema 2: `NullPointerException` ao `sendAlert` por email
O sistema lança uma exceção `NullPointerException` ao tentar enviar alertas por email, indicando que um objeto necessário não foi inicializado corretamente.

###### Erro

```
2020-10-20 13:02:14,575 WARN [c.c.a.AlertManagerImpl] (Cluster-Notification-1:ctx-271aae8f) (logid:61ec79c9) AlertType:: 14 | dataCenterId:: 0 | podId:: 0 | clusterId:: null | message:: Management server node 172.16.202.101 is up
2020-10-20 13:02:14,595 ERROR [c.c.a.AlertManagerImpl] (Cluster-Notification-1:ctx-271aae8f) (logid:61ec79c9) Problem sending email alert
java.lang.NullPointerException
at com.cloud.alert.AlertManagerImpl$EmailAlert.sendAlert(AlertManagerImpl.java:790)
at com.cloud.alert.AlertManagerImpl.sendAlert(AlertManagerImpl.java:248)
```

Método impactado:`sendAlert` em `AlertManagerImpl.java`

Problema: O objeto `_emailAlert` está nulo quando o método `sendAlert` é chamado.

Possíveis Causas:
  - Falha na inicialização do objeto `_emailAlert`.
  - Configurações de email (Simple Mail Transfer Protocol) ausentes ou incorretas.

Soluções sugeridas:

- Na inicialização do objeto verificar se o objeto `_emailAlert` é corretamente inicializado antes do uso.
- Confirmar que as configurações SMTP estão definidas corretamente.
- Adicionar logs mais detalhados para capturar informações adicionais durante o envio de email.
- Implementar verificações de nulo adicionais para evitar exceções.

###### [Código principal](https://github.com/apache/cloudstack/blob/4.13/server/src/main/java/com/cloud/alert/AlertManagerImpl.java#L746)

``` 
public void sendAlert(AlertType alertType, long dataCenterId, Long podId, String subject, String body) {

        // publish alert
        AlertGenerator.publishAlertOnEventBus(alertType.getName(), dataCenterId, podId, subject, body);

        // TODO:  queue up these messages and send them as one set of issues once a certain number of issues is reached?  If that's the case,
        //         shouldn't we have a type/severity as part of the API so that severe errors get sent right away?
        try {
            if (_emailAlert != null) {
                _emailAlert.sendAlert(alertType, dataCenterId, podId, null, subject, body);
            } else {
                s_logger.warn("AlertType:: " + alertType + " | dataCenterId:: " + dataCenterId + " | podId:: " + podId +
                        " | message:: " + subject + " | body:: " + body);
            }
        } catch (Exception ex) {
            s_logger.error("Problem sending email alert", ex);
        }
    }
```

###### Conclusão

Nos dois problemas, revisar a inicialização dos objetos e a configuração do ambiente, que logs adicionais sejam usados para capturar detalhes em tempo de execução e usar de testes unitários para capturar melhor o problema e evitar futuros erros.
