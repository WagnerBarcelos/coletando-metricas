# coletando-metricas

# Pré-requisitos

- .NET SDK
- Docker
- Prometheus
- Grafana

# Passo a Passo

## 1. Criar o Aplicativo ASP.NET Core

Crie um novo aplicativo ASP.NET Core e adicione os pacotes necessários:

````
dotnet new web -o WebMetric
cd WebMetric
dotnet add package OpenTelemetry.Exporter.Prometheus.AspNetCore --prerelease
dotnet add package OpenTelemetry.Extensions.Hosting
````

Substitua o conteúdo de Program.cs pelo seguinte código:

```
using OpenTelemetry.Metrics;

var builder = WebApplication.CreateBuilder(args);
builder.Services.AddOpenTelemetry()
    .WithMetrics(builder =>
    {
        builder.AddPrometheusExporter();

        builder.AddMeter("Microsoft.AspNetCore.Hosting",
                         "Microsoft.AspNetCore.Server.Kestrel");
        builder.AddView("http.server.request.duration",
            new ExplicitBucketHistogramConfiguration
            {
                Boundaries = new double[] { 0, 0.005, 0.01, 0.025, 0.05,
                       0.075, 0.1, 0.25, 0.5, 0.75, 1, 2.5, 5, 7.5, 10 }
            });
    });
var app = builder.Build();

app.MapPrometheusScrapingEndpoint();

app.MapGet("/", () => "Hello OpenTelemetry! ticks:"
                     + DateTime.Now.Ticks.ToString()[^3..]);

app.Run();
```

## 2. Exibir Métricas com dotnet-counters

Instale a ferramenta dotnet-counters:

````
dotnet tool update -g dotnet-counters
````

Inicie o aplicativo de teste e, em seguida, execute:

```
dotnet-counters monitor -n WebMetric --counters Microsoft.AspNetCore.Hosting
```

### 3. Configurar Prometheus

Modifique o arquivo prometheus.yml para incluir o endpoint do aplicativo ASP.NET Core:

```
scrape_configs:
  - job_name: 'MyASPNETApp'
    scrape_interval: 5s
    static_configs:
      - targets: ["localhost:5045"]
```

## 4. Iniciar Prometheus
Recarregue a configuração ou reinicie o servidor Prometheus. Confirme que o estado está UP na página de Status do Prometheus.

## 5. Configurar Grafana
Instale o Grafana e configure uma nova fonte de dados apontando para o Prometheus. Crie um novo painel para visualizar as métricas.

## 6. Criar Métricas Personalizadas
Para criar métricas personalizadas, adicione um contador ao seu projeto. No arquivo Program.cs, adicione:

```
public class ContosoMetrics
{
    private readonly Counter<int> _productSoldCounter;

    public ContosoMetrics(IMeterFactory meterFactory)
    {
        var meter = meterFactory.Create("Contoso.Web");
        _productSoldCounter = meter.CreateCounter<int>("contoso.product.sold");
    }

    public void ProductSold(string productName, int quantity)
    {
        _productSoldCounter.Add(quantity,
            new KeyValuePair<string, object?>("contoso.product.name", productName));
    }
}

```
## 7. Registro de Métricas no Program.cs
Registre o tipo de métrica com DI em Program.cs:

```
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<ContosoMetrics>();
```

## 8. Inserir Tipo de Métrica
Insira o tipo de métrica e registre os valores quando necessário:

```
app.MapPost("/complete-sale", (SaleModel model, ContosoMetrics metrics) =>
{
    // Lógica de negócios, como salvar a venda no banco de dados...

    metrics.ProductSold(model.ProductName, model.QuantitySold);
});
```

## 9. Testar Métricas em Aplicativos do ASP.NET Core
Para testar as métricas, utilize MetricCollector<T> em seus testes de integração:

```
public class BasicTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;
    public BasicTests(WebApplicationFactory<Program> factory) => _factory = factory;

    [Fact]
    public async Task Get_RequestCounterIncreased()
    {
        // Arrange
        var client = _factory.CreateClient();
        var meterFactory = _factory.Services.GetRequiredService<IMeterFactory>();
        var collector = new MetricCollector<double>(meterFactory,
            "Microsoft.AspNetCore.Hosting", "http.server.request.duration");

        // Act
        var response = await client.GetAsync("/");

        // Assert
        Assert.Contains("Hello OpenTelemetry!", await response.Content.ReadAsStringAsync());

        await collector.WaitForMeasurementsAsync(minCount: 1).WaitAsync(TimeSpan.FromSeconds(5));
        Assert.Collection(collector.GetMeasurementSnapshot(),
            measurement =>
            {
                Assert.Equal("http", measurement.Tags["url.scheme"]);
                Assert.Equal("GET", measurement.Tags["http.request.method"]);
                Assert.Equal("/", measurement.Tags["http.route"]);
            });
    }
}
```

## Evidências

Os prints com as evidências estão na pasta /assets do projeto.




