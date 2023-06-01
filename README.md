# Authorize WebAPI with Auth0

This example shows how to authenticate a user using Auth0

## Run the Project

Update the `appsettings.json` with your Auth0 settings:

```json
{
  "Auth0": {
    "Domain": "Your Auth0 domain",
    "Audience": "Your Auth0 Client Id"
  } 
}
```

Restore the NuGet packages and run the application: (Skip this if you are using Visual Studio)


```bash
dotnet restore

dotnet run
```

Once the app is up and running, verify that the public API endpoint can be reached, either by using `curl` or accessing `http://localhost:3010/api/public` in the browser. You should see the following output:

```json
{
  "message": "Hello from a public endpoint! You don't need to be authenticated to see this."
}
```

## Run this project with Docker (Optional)

In order to run the example with Docker you need to have [Docker](https://docker.com/products/docker-desktop) installed.

To build the Docker image and run the project inside a container, run the following command in a terminal, depending on your operating system:

```
# Mac
sh exec.sh

# Windows (using Powershell)
.\exec.ps1
```
## Define Permissions

You can define allowed permissions in the Permissions view of the Auth0 APIs section [Applications -> APIs -> Permissions -> Add a Permission(Scope)].

This example uses the read:dashboards scope.

After that on Machine to Machine Applications Tab you will check Athorized toggled and grant a permission to a client(app)

Note: Check Docs Folder inside the project.

## Calling the API

### 1. Obtain an access token

To access the secure endpoint you will need to [obtain an access token](https://auth0.com/docs/secure/tokens/access-tokens/get-access-tokens)   requesting an access token when authenticating a user.

From csharp using Resharp nuget:
```csharp
var client = new RestClient("https://{yourDomain}/oauth/token");
var request = new RestRequest();
request.AddHeader("content-type", "application/x-www-form-urlencoded");
request.AddParameter("application/x-www-form-urlencoded", "grant_type=client_credentials&client_id=%24%7Baccount.clientId%7D&client_secret=YOUR_CLIENT_SECRET&audience=%24%7BapiIdentifier%7D", ParameterType.RequestBody);
var response = client.Execute(request,Method.Post);
```

From PostMan:

Method->Post
URL: https://{yourDomain}/oauth/token

Request
Body->raw
```json
{"client_id":"yourCliendID","client_secret":"yourClientSecret","audience":"yourAudience","grant_type":"yourGrantType"}
```

Response
```json
{
    "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6ImVoUi1WZ1RjdzRZT1lWX21tMFhVdiJ9..z6kAEF58sc7XlCyk40f6YZgXTXbKd2KDNqprjDO6NFvRFLgSq8DGMdFo4w4Q2zEJ6jF8_m0qlFfV7jc25KacNq2-7abktRqm34vhM5OXDssRmm0Owin6LETIp0OtyMbZlz7lsr4n-OjihvJy65wYcagc1OYkHTeFcpTfxKUQNGdpeZxXreSR4cZH276IeGtgBJ_mXgIqxuWWGrhIuj3kUpcba3VfII9YRagrJ2JpP5wmMm4xStIFxxB5B6s61ea6rfZwp0whigs51iVi6lfYnoBp6rsbQB1r-RaW0jCx7ALGtyvLwLkToEh1fuY-uOlm2XeCrWlKQNtv22CMBs1P4A",
    "scope": "read:dashboards read:messages",
    "expires_in": 86400,
    "token_type": "Bearer"
}
```

The access token is a JWT a you will see something just like this in the payload:

```json
{
  "iss": "https://yourDomain.com/",
  "sub": "qOWSbLuqV9uitKBk1PWwtfOCROXGmoSH@clients",
  "aud": "https://yourDomain.com",
  "iat": 1685579536,
  "exp": 1685665936,
  "azp": "qOWSbLuqV9uitKBk1PWwtfOCROXGmoSH",
  "scope": "read:dashboards read:messages",
  "gty": "client-credentials",
  "permissions": [
    "read:dashboards",
    "read:messages"
  ]
}
```

Note: You will get permissions scopes in the payload of the JWT if you check "Add permissions in the access token" [API Settings -> RBAC Settings]

### 1. Call a secure endpoint

When a user authenticates, you request an access token then pass the access token as a **Bearer** token in the **Authorization** header when calling the `http://localhost:3010/api/private` endpoint.

From Postman:

Request
Body->x-www-form-urlencoded

Key: application/x-www-form-urlencoded
Value:  grant_type=client_credentials&client_id=%24%7Baccount.clientId%7D&client_secret=YOUR_CLIENT_SECRET&audience=%24%7BapiIdentifier%7D

Response
```json
{
    "message": "Hello from a private endpoint! You need to be authenticated and have a scope of read:dashboards to see this."
}
```

## Important Snippets

### 1. Register Authentication Services & CORS policy

```csharp
// Startup.cs

public void ConfigureServices(IServiceCollection services)
{
    // Leave any code your app/template already has here and just add these lines:
    services.AddCors(options =>
	{
		options.AddPolicy("AllowSpecificOrigin",
			builder =>
			{
				builder
				.WithOrigins("http://localhost:8080")
				.AllowAnyMethod()
				.AllowAnyHeader()
				.AllowCredentials();
			});
	});
	
    var domain = $"https://{Configuration["Auth0:Domain"]}/";
    services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
		.AddJwtBearer(options =>
		{
			options.Authority = domain;
			options.Audience = Configuration["Auth0:Audience"];
		});
}
```

### 2. Register Authentication Middleware & Enable CORS

```csharp
// Startup.cs

public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
{
    // Leave any code your app/template already has here and just add the line:
    app.UseCors("AllowSpecificOrigin");
	app.UseAuthentication();
	app.UseAuthorization();
}
```

### 3. Secure an API method

```csharp
// /Controllers/ApiController.cs

[Route("api/[controller]")]
[ApiController]
public class YourController : ControllerBase
{
    [HttpGet]
    [Authorize]
    public IActionResult Private()
    {
        return Ok(new
        {
            Message = "Hello from a private endpoint! You need to be authenticated to see this."
        });
    }
}
```

