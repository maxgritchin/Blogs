# Продвинутые способы .NET конфигурации

Каждое приложение имеет несколько источников хранения настроек. Обсудим с вами как читать данные из разных источников, группировать их для использования внутри приложения и разбираться в конфигурации.

## IConfiguration 

`IConfiguration` — это набор свойств конфигурации приложения в виде пар "ключ — значение". интерфейс `IConfiguration` может быть реализован для чтения из разных источников данных.

![IConfiguration](img/i_config.png?raw=true "IConfiguration")

Конфигурация происходит путем указания из каких источников стоит читать данные.
При этом каждый поставщик конфигурации добавляет “слой” значений к ConfigurationBuilder. Метод Build() прочтет конфигурацию из указаного списка ичточников. Причем, последовательность указания источников данных имеет значени. Кажду последующий ичтоник переопределить значение с таким же ключем, которое было в предыдущем источнике. 

![Configuration overriding](img/config_override.png?raw=true "Configuration overriding")

Для определения списка источников данных используется класс `ConfigurationBuilder`. Метод `Build()` сделает чтение данных и вернет объект `IConfiguration` через который осуществляется доступ к данным. 

При запуске приложения необходимо настроить чтение данных из разных источников.
По умолчанию присутствует поддержка JSON, environments, CLI arguments и другие.

![List config sources](img/list_config_src.png?raw=true "List config sources")

Например, указан такой порядок: `AddJsonFile`, затем `AddEnvironmentsVariables`, то вначале будут прочитаны данные из JSON файла, а затем — переменных окружения. Если в переменной окружения есть какое-либо значение, то информация JSON файла по ключу будет перезаписана. Метод `Build()` при этом возвращает `IСonfiguration` объект, который впоследствии можно использовать для чтения данных.

Непосредственно для чтения данных используем следующий код:

```csharp
public class MySettings
{
    public string AppName { get; set; }
    public int Port { get; set; }
    public string Host { get; set; }
}

// десериализовать всю секцию
var settings = configuration.GetRequiredSection("MySettings").Get<MySettings>();

// получить отдельное значение
var port = configuration.GetValue<int>("MySettings:Port");

// строку
var host = configuration["MySettings:Host"];
```

### Что делать если источник не стандартный?

Если ни один из доступных поставщиков данных вам не подходит, покажем как реализовать настраиваемого поставщика данных. Например, использовать вашу базу данных или данные из объекта как источник конфигурации.

Создадим собственное расширение для чтения конфигурации. Специальный поставщик для чтения параметров из предоставленного `Dictionary` объекта.

1. Provider (чтение/запись данных)

```csharp
public class DictionaryConfigurationProvider : ConfigurationProvider
{
    private readonly IDictionary<string, string> _data;

    public DictionaryConfigurationProvider(IDictionary<string, string> data)
        => _data = data;
    
    public override void Load() => Data = _data;
}
```

2. Source (интеграция Provider и Builder)

```csharp
public class DictionaryConfigurationSource : IConfigurationSource
{
    private readonly IDictionary<string, string> _data;

    public DictionaryConfigurationSource(IDictionary<string, string> data) 
        => _data = data;
    
    public IConfigurationProvider Build(IConfigurationBuilder builder) 
        => new DictionaryConfigurationProvider(_data);
}
```

3. Расширение для `IConfigurationBuilder`

```csharp
public static class ConfigurationBuilderExtensions
{
    public static IConfigurationBuilder AddSimpleConfiguration(
    
    this IConfigurationBuilder builder, IDictionary<string, string> data)
    {
        data ??= new Dictionary<string, string>();
        return builder.Add(new DictionaryConfigurationSource(data));
    }
}
```

4. И последний шаг

```csharp
var data = new Dictionary<string, string>
{
{ "Hello", "World" }
};

var configuration = new ConfigurationBuilder()
    .AddSimpleConfiguration(data)
    .Build();

var who = configuration["Hello"];
Console.WriteLine($"Hello, {who}");

// ---
// Expected Result -> Hello, World
```

### Продвинутая конфигурация для использования в тестах

Для проведения интеграционных или функциональных тестов часто приходится использовать реальное подключение к базе данных. Для этого подходит `IConfiguration` — делаем подстановку переменных (через environment variables) без необходимости сохранять данные в репозитории приложения.

Модульные тесты в Visual Studio можно настраивать с помощью файла `.runsettings`. Например, вы можете изменить версию .NET, на которой выполняются тесты, каталог для результатов тестирования или данные, которые собираются во время выполнения теста. Обычно файл `.runsettings` используется для настройки анализа покрытия кода.

Файлы параметров запуска можно использовать для настройки тестов, запускаемых из командной строки, из среды IDE или в рабочем процессе сборки с помощью Azure Test Plans или Team Foundation Server (TFS).

Файлы настроек запуска необязательны. Если вам не требуется особая конфигурация, вам также не нужен файл `.runsettings`.

1. Для использования продвинутой конфигурации в тестах создаем локальный файл .runsettings. Тут будет указана переменная окружения `DB_CONNECTION_STRING`, которая может быть прочитана при запусках тестов.

```xml
<?xml version="1.0" encoding="utf-8"?>
<!-- File name extension must be .runsettings -->
<RunSettings>
    <RunConfiguration>
        <EnvironmentVariables>
            <DB_CONNECTION_STRING>
                Server=server;Database=db;Port=5432;UserId=user;Password=pass;
            </DB_CONNECTION_STRING>
        </EnvironmentVariables>
    </RunConfiguration>
</RunSettings>
```

2. Создаем путь к файлу через csproj
<Project Sdk="Microsoft.NET.Sdk">
    <PropertyGroup>
        <RunSettingsFilePath>$(MSBuildProjectDirectory)\example.runsettings</RunSettingsFilePath>
    </PropertyGroup>
    ...
</Project>

3. ИЛИ указываем в IDE (в примере Rider)

![Rider config](img/rider_config.png?raw=true "Rider Config")

#### Использование в тестах

```csharp
private const string DB_CS_ENV = "DB_CONNECTION_STRING";
private PostgresConnection _connection;

[OneTimeSetUp]
public void Setup()
{
    var config = new ConfigurationBuilder()
        .AddEnvironmentVariables()
        .Build();
    _connection = new PostgresConnection(config.GetSection(DB_CS_ENV).Value);
}

/*
HERE LOGIC FOR TESTS
...
*/

```