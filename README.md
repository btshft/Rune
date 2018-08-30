## Идея
Создать библиотеку, которая бы позволяла в рантайме генерировать динамические классы-клиенты для работы с HTTP REST сервисами как с обычным C# классом. Похожим примером могут служить клиенты WCF сервисов из .NET Framework.

## Мотивация
Часто взаимодействие с внешними сервисами представляет собой приблизительно следующий сценарий
1. Создается класс-обертка, инкапсулирующий в себе работу с HttpClient
2. В классе реализуются однотипные методы обращений к сервисам, где происходит примерно следующее 
   1) В блоке using инициализируется экземпляр HttpClient (или используется его статическая версия)
   2) Устанавлиются параметры клиента - базовый адрес, таймайт, заголовки, адрес метода и пр. 
   3) Выполняется сериализация Payload (QueryString, Body)
   4) Выполняется запрос (PostAsync, GetAsync, etc)
   5) Выполняется проверка статус кода ответа и десериализация ответа от сервиса
3. В необходимых местах вызываются методы данного класса для взаимодействия с внешней системой

Упрощенный сценарий можно представить так:
```csharp

public class CustomerServiceFacade {

    private readonly HttpClient _client;

    public CustomerServiceFacade() {
        _client = new HttpClient();
    }

    public async Task<List<Customer>> GetCustomers(int marketId) {
        var url = $"{Configuration.GetCustomersUrl}/{marketId}";
        using (var response = await _client.GetAsync(url)) 
        {
            response.EnsureSuccessStatusCode();
            return response.Content.ReadAsAsync<List<Customer>>();
        }
    }
    
    // И ещё миллион методов подобным тому, что выше
}
```
За счет библиотеки можно избавиться от дублирования кода и улучшить детализированность контракта взаимодействия.

## Черновик API
В качестве основного API выступает интерфейс описываемый пользователем библиотеки и являющийся контрактом взаимодействия с внешней системой. Например, в случае рассмотренного выше сервиса заказчиков, контракт можно представить следующим образом

```csharp
public interface ICustomerHttpService {
    Task<List<Customer>> GetCustomers(int marketId);
}
```

Однако данное описание не содержит достаточной информации для взаимодействия с сервисом. Поэтому необходимо доработать контракт метаданными. К метаданным можно отнести следующее

Базовые
1) Базовый адрес сервиса
2) Адрес метода
3) Тип HTTP метода (verb)

Дополнительные
1) Способ передачи параметров (?)
2) Заголовки (в т.ч. авторизация)
3) Время ожидания ответа (Timeout)
4) Куки

Базовые метаданные имеет смысл конфигурировать в виде атрибутов, располагающихся на интерфейсе и метода. 
1) ```[ServiceEndpoint("{BaseUrl}")]``` - размещается на интерфейсе, содержит базовый адрес сервиса при вызове методов
2) ```[MethodEndpoint("{MethodUrl}", {Verb})]``` - размещается на методе интерфейса, содержит относительный адрес метода. Также для данного атрибута имеет смысл создать сокращения ```[PostMethodEndpoint("...")]```, ```[GetMethodEndpoint("...")]```.
3) Способ передачи параметров ```[BodyParameter]``` и ```[QueryParameter]```.

Таймауты, заголовки и авторизацию думаю имеет смысл конфигурировать уже на уровне фабрики клиента, чтобы не перегружать контракт сервиса атрибутами. Или же подумать как это сделать на уровне атрибутов.

Исходя из изложенного, можно переписать контракт на следующий
```csharp
[ServiceEndpoint("https:\\customer.service\api\v1")]
public interface ICustomerHttpService {
    [GetMethodEndpoint("customers\{marketId}")]
    Task<List<Customer>> GetCustomers(int marketId);
}
```

Создание самого клиента предполагается сделать через фабрику, по типу следующего:
```csharp
public interface IHttpServiceClientFactory<TService> {
    TService CreateClient();
}
```
Либо же простой сценарий создания "на месте" через обращение к инициализированной статической фабрики библиотеки
```csharp
public static class Rune {
    public static TClient CreateClient<TClient>(IConfiguration config);
    public static TClient CreateClient<TClient>();
}
```

## Вопросы / Проблемы / Идеи
1. Что должно конфигурироваться? 
   1) Сериализатор параметров/ответа
   2) Сервис резолва параметров в URI. Чтобы можно было писать ```[ServiceEndpoint("{ServiceUri}/api")]```, а значение ключа ```{ServiceUri}``` подставлять из настроек приложения / другого места.
2. Механизмы авторизации
3. Установка cookie в запрос
4. Логирование
5. Разделение скоупа настроек (глобальные, service-specific, method-specific)
6. Сериализация/Десериализация (Newtonsoft.Json?)
7. Механизм создания прокси типов (Castle.Core или написать свой)
8. Обработка и прокидывание исключений
9. Разделение инстансов HttpClient-ов. Может сделать пул? Конфигурация 1 сервис - 1 HttpClient?
10. Хедеры в запросах. Как передавать?
11. Разрешать ли синхронные вызовы, или только асинхронные?
12. Тип возвращаемого значения метода ? Task<T>, T или CustomTask<T>. 
13. Может ли вообще пригодиться CustomTask<T>?
14. Проработать API создания экземпляров сервисов. Может нужна метафабрика? 
15. Подумать над типичными сценариями использования
16. Сериализация параметров по умолчанию. GEQ -> QueryString, Post -> Body
