
# Why an API?

**Objective:** Understand the reasons for creating an API with ASP.NET Core and the benefits it provides.

## Key Concepts

1. **ASP.NET Core:**
    - Part of the .NET ecosystem, ASP.NET Core is a powerful framework for building web applications and APIs.
    - The web portion of .NET is still called ASP.NET Core, even though .NET Core is now simply .NET.

2. **Uniform Contract:**
    - APIs provide a consistent way for different clients to communicate with the server.
    - Clients can include web applications, mobile apps, native applications, and desktop applications.
    - A uniform contract ensures that all clients use the same language and format to communicate.

3. **Data Sharing:**
    - APIs allow for sharing data and information across different platforms and devices.
    - They enable seamless integration and data exchange between the server and multiple clients.

## Benefits of Using an API

### 1. **Decoupling:**
    - APIs decouple the client and server, allowing them to evolve independently.
    - Clients only need to know how to communicate with the API, not the internal workings of the server.

### 2. **Scalability:**
    - APIs support scalability by enabling multiple clients to access the same data source.
    - They can handle increasing loads by distributing requests across servers.

### 3. **Reusability:**
    - APIs promote reusability by providing a standard way to access functionality.
    - Different applications can reuse the same API to perform similar tasks.

### 4. **Flexibility:**
    - APIs provide flexibility in development and deployment.
    - They allow for integrating various technologies and platforms.

### 5. **Standardization:**
    - APIs adhere to standards such as REST (Representational State Transfer), ensuring consistency and compatibility.
    - Standardization simplifies development and maintenance.

## Implementing an API with ASP.NET Core

### 1. **Setting Up the Project:**
    - Create a new ASP.NET Core project for the API.
    - Configure the project to use .NET 8, the latest long-term support version.

### 2. **Defining Endpoints:**
    - Use controllers or minimal APIs to define endpoints.
    - Map endpoints to specific routes and HTTP methods.

### 3. **Handling Requests and Responses:**
    - Implement logic to handle incoming requests and generate appropriate responses.
    - Use data models and serialization to format the data.

### 4. **Security and Authentication:**
    - Implement security measures such as authentication and authorization.
    - Ensure that only authorized clients can access the API.

### 5. **Testing and Documentation:**
    - Test the API using tools like Swagger or Postman.
    - Document the API to provide clear instructions for clients on how to use it.

## Example Code

```csharp
using Microsoft.AspNetCore.Builder;
using Microsoft.AspNetCore.Mvc;
using Microsoft.EntityFrameworkCore;
using Microsoft.Extensions.DependencyInjection;
using System.Collections.Generic;
using System.Linq;
using System.Threading.Tasks;

var builder = WebApplication.CreateBuilder(args);

builder.Services.AddDbContext<ShopContext>(options =>
    options.UseSqlServer(builder.Configuration.GetConnectionString("DefaultConnection")));

var app = builder.Build();

using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<ShopContext>();
    await db.Database.EnsureCreatedAsync();
}

app.MapGet("/products", async (ShopContext _context) =>
{
    return await _context.Products.ToListAsync();
});

app.MapGet("/products/{id}", async (int id, ShopContext _context) =>
{
    var product = await _context.Products.FindAsync(id);
    if (product == null)
    {
        return Results.NotFound();
    }
    return Results.Ok(product);
});

app.Run();
```

## Key Points

- APIs provide a uniform way for different clients to communicate with the server.
- ASP.NET Core is a powerful framework for building APIs.
- APIs enable data sharing, decoupling, scalability, reusability, flexibility, and standardization.
- Implementing an API involves setting up the project, defining endpoints, handling requests and responses, ensuring security, and testing and documenting the API.

By understanding these concepts and benefits, you'll be better prepared to create effective and efficient APIs with ASP.NET Core.
