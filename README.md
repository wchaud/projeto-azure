Exemplo completo e prático de um projeto integrado usando os serviços da Azure:

Azure Container Apps: hospeda uma aplicação em container (ex: API backend ou processador).

Azure Functions: executa uma lógica leve sob demanda (ex: envio de e-mails).

Azure Service Bus (intermediário): envia uma mensagem da aplicação containerizada para a Azure Function.

No final, a Azure Function envia um e-mail usando, por exemplo, o SendGrid.

Arquitetura resumida

Usuário ou Sistema
      ↓
Azure Container App (ex: app .NET ou Node)
      ↓
Service Bus Queue
      ↓
Azure Function (gatilhada pela fila)
      ↓
SendGrid API (envia e-mail)

Tecnologias usadas

Azure Container App (com imagem Docker)

Azure Service Bus (Queue)

Azure Function (gatilho de Service Bus)

SendGrid (para envio de e-mail)

Passo a passo (resumo)

App containerizado que publica na fila

// EnviarMensagemService.cs (exemplo em C#)
public class EnviarMensagemService
{
    private readonly ServiceBusClient _client;
    private readonly ServiceBusSender _sender;

    public EnviarMensagemService(string connectionString, string queueName)
    {
        _client = new ServiceBusClient(connectionString);
        _sender = _client.CreateSender(queueName);
    }

    public async Task EnviarAsync(string emailDestino, string mensagem)
    {
        var payload = new { email = emailDestino, conteudo = mensagem };
        var json = JsonSerializer.Serialize(payload);

        await _sender.SendMessageAsync(new ServiceBusMessage(json));
    }
}

 Azure Function com gatilho de Service Bus

public class EmailFunction
{
    private readonly ISendGridClient _sendGridClient;

    public EmailFunction(ISendGridClient sendGridClient)
    {
        _sendGridClient = sendGridClient;
    }

    [Function("EnviarEmail")]
    public async Task RunAsync(
        [ServiceBusTrigger("minha-fila", Connection = "ServiceBusConnection")] string myQueueItem,
        FunctionContext context)
    {
        var dados = JsonSerializer.Deserialize<EmailPayload>(myQueueItem);

        var msg = new SendGridMessage()
        {
            From = new EmailAddress("seuemail@dominio.com", "Sistema"),
            Subject = "Notificação",
            PlainTextContent = dados.Conteudo,
            HtmlContent = $"<strong>{dados.Conteudo}</strong>"
        };
        msg.AddTo(new EmailAddress(dados.Email));

        await _sendGridClient.SendEmailAsync(msg);
    }
}

public class EmailPayload
{
    public string Email { get; set; }
    public string Conteudo { get; set; }
}

Deploy com Azure CLI

az group create --name meu-grupo --location brazilsouth

# Service Bus
az servicebus namespace create --name meu-servicebus-ns --resource-group meu-grupo --location brazilsouth
az servicebus queue create --name minha-fila --namespace-name meu-servicebus-ns --resource-group meu-grupo

Azure Container App:

az containerapp create \
  --name meu-app-container \
  --resource-group meu-grupo \
  --image meucontainer.azurecr.io/meuapp:latest \
  --environment meu-ambiente \
  --ingress external \
  --target-port 80 \
  --env-vars ServiceBusConnection="Endpoint=sb://..."

Azure Function:

az functionapp create \
  --resource-group meu-grupo \
  --consumption-plan-location brazilsouth \
  --runtime dotnet \
  --functions-version 4 \
  --name minha-func-email \
  --storage-account meuarmazenamento \
  --app-insights-key <chave>

az functionapp config appsettings set \
  --name minha-func-email \
  --resource-group meu-grupo \
  --settings ServiceBusConnection="Endpoint=sb://..." SendGridApiKey="..."



Resultado

Quando a aplicação no Container App executa sua lógica e chama o EnviarMensagemService, a mensagem vai para o Service Bus. Assim que ela entra na fila, a Azure Function é automaticamente executada e envia o e-mail usando o SendGrid.




 




