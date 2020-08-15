# Azure.Functions.WebApi

This project allows developers to write Azure Functions HTTP Triggers in a familiar ASP.NET controller-style syntax, rather than the verbose and unwieldy parameter-attribute trigger syntax employed by Azure Functions.

The user code is scanned for classes deriving from `ApiFuncController` and code-gens the necessary Azure Function HTTP Trigger endpoints.

Note that this is *not* intended to be a complete, drop-in replacement for ASP.NET MVC, Web Api or ASP.NET Core - this in intended to provide absolute basics to make transitioning small and simple APIs to Azure Functions easier, not to provide a fully compatible conversion path.

## Example Codegen

```csharp

[RoutePrefix("status")]
public partial class StatusController : ApiFuncController
{
    [Inject] private IStatusService StatusService { get; }

    [HttpGet("")]
    [EnableQuery(MaxTop = 100, AllowedQueryOptions = AllowedQueryOptions.Select | AllowedQueryOptions.Filter | AllowedQueryOptions.Top)]
    public async Task<IQueryable<Status>> GetAll() => await StatusService.GetAllStatuses();

    [HttpGet("{customer}")]
    public async Task<Status> Get(string customer) => await StatusService.GetCustomerStatus(customer);
    
    [HttpPost("{customer}"), Authorize]
    public async Task Set(string customer, CustomerStatus status)
    {            
        await StatusService.SetCustomerStatus(customer, status);
    }
}

```

```csharp

partial class StatusController
{
    public StatusController(IStatusService statusService)
    {
        StatusService = statusService;
    }
}

public sealed class StatusControllerFunctions
{
    private StatusController Controller { get; }
    private AuthenticationHandler AuthenticationHandler { get; }
    
    public StatusControllerFunctions(StatusController controller, AuthenticationHandler authenticationHandler)
    {
        Controller = controller;
        AuthenticationHandler = authenticationHandler;
    }
    
    [FunctionName("StatusController-GetAll")]
    [EnableQuery(MaxTop = 100, AllowedQueryOptions = AllowedQueryOptions.Select | AllowedQueryOptions.Filter | AllowedQueryOptions.Top)]
    public async Task<IActionResult> GetAll(
        [HttpTrigger(Anonymous, "get", Route = "status")] HttpRequest req,
        ODataQueryOptions<Status> odata
    )
    {
        var result = await Controller.GetAll();
        var oDataResult = odata.ApplyTo(result);
        return new OkObjectResult(oDataResult);
    }
    
    [FunctionName("StatusController-Get-Set")]
    public async Task<IActionResult> Get-Set([HttpTrigger(Anonymous, "get", "post", Route="status/{customer}")] HttpRequest req, string customer)
    {
        var identity = await AuthenticationHandler.Auth(req);
        if (identity is null)
            return new UnauthorizedResult();
    
        Controller.SetRequestProperties(req, identity);
    
        if (req.Method == "get")
        {
            return new OkObjectResult(await Controller.Get(customer));
        }
        else if (req.Method == "post")
        {
            var bodyObject = req.ReadBodyAsJson<CustomerStatus>();
            
            if (!bodyObject.ValidateObject())
                return new BadRequestResult();
                
            await Controller.Set(customer, bodyObject);
            
            return new NoContentResult();
        }
        else
        {
            throw new InvalidOperationException("Invalid HTTP Method");
        }
    }
}

```

## Motivation

Converting older ASP.NET Web API projects to Azure Functions has been a bit painful for me, as while you can relatively simple convert from one to the other (assuming you're not doing lots of custom things!), Functions HTTP Triggers can tend to end up as an ugly, ugly mess, as compared to MVC's controllers.

After doing this a number of times, I got tired of having to change a lot of this code for Functions by hand and making it uglier, and hard to code-review the change for my colleagues. Most of the controllers I was rewriting could basically be converted by rote, and had no complicated requirements around serialization or authorization, and didn't look like they really needed to change if a Function effectively wrapped it, rather than replaced it.