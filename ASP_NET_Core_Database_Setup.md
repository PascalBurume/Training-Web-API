
# ASP.NET Core Database Setup for SEGUCE and DGDA Data

## Étape 1 : Créer un nouveau projet ASP.NET Core
1. Ouvrez Visual Studio.
2. Créez un nouveau projet ASP.NET Core Web Application.
3. Choisissez un modèle "API" ou "MVC" selon vos besoins.

## Étape 2 : Configurer Entity Framework Core
1. Installez les packages NuGet nécessaires :
    ```bash
    dotnet add package Microsoft.EntityFrameworkCore
    dotnet add package Microsoft.EntityFrameworkCore.SqlServer
    dotnet add package Microsoft.EntityFrameworkCore.Tools
    dotnet add package Microsoft.AspNetCore.Mvc.NewtonsoftJson
    ```

## Étape 3 : Définir les modèles de données
Créez des classes de modèle pour représenter les données SEGUCE, DGDA et les mouvements d'entrée OCC.

```csharp
public class SeguceData
{
    public int Id { get; set; }
    public string XmlData { get; set; }
    public string ExcelData { get; set; }
    public DateTime DateCreated { get; set; }
}

public class DgdaData
{
    public int Id { get; set; }
    public string XmlData { get; set; }
    public string ExcelData { get; set; }
    public DateTime DateCreated { get; set; }
}

public class OccMovement
{
    public int Id { get; set; }
    public string MovementDetails { get; set; }
    public DateTime Date { get; set; }
    public int SeguceDataId { get; set; }
    public SeguceData SeguceData { get; set; }
}
```

## Étape 4 : Créer le contexte de base de données
Définissez le contexte de la base de données pour utiliser Entity Framework Core.

```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<SeguceData> SeguceData { get; set; }
    public DbSet<DgdaData> DgdaData { get; set; }
    public DbSet<OccMovement> OccMovements { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);
        
        // Configurations supplémentaires si nécessaire
    }
}
```

## Étape 5 : Configurer la chaîne de connexion
Ajoutez la chaîne de connexion à votre fichier `appsettings.json`.

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=(localdb)\mssqllocaldb;Database=YourDatabaseName;Trusted_Connection=True;MultipleActiveResultSets=true"
  }
}
```

## Étape 6 : Enregistrer le contexte de base de données dans le conteneur de services
Ajoutez le contexte de base de données dans le fichier `Startup.cs`.

```csharp
public class Startup
{
    public void ConfigureServices(IServiceCollection services)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));
        
        services.AddControllers()
            .AddNewtonsoftJson(options =>
                options.SerializerSettings.ReferenceLoopHandling = Newtonsoft.Json.ReferenceLoopHandling.Ignore);
        
        // Autres configurations
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        // Configuration du middleware
    }
}
```

## Étape 7 : Créer des migrations et mettre à jour la base de données
Exécutez les commandes suivantes pour créer des migrations et mettre à jour la base de données.

```bash
dotnet ef migrations add InitialCreate
dotnet ef database update
```

## Étape 8 : Créer des contrôleurs pour gérer les données
Créez des contrôleurs API pour gérer les opérations CRUD pour vos tables.

```csharp
[ApiController]
[Route("api/[controller]")]
public class SeguceDataController : ControllerBase
{
    private readonly ApplicationDbContext _context;

    public SeguceDataController(ApplicationDbContext context)
    {
        _context = context;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<SeguceData>>> GetSeguceData()
    {
        return await _context.SeguceData.ToListAsync();
    }

    [HttpPost]
    public async Task<ActionResult<SeguceData>> PostSeguceData(SeguceData seguceData)
    {
        _context.SeguceData.Add(seguceData);
        await _context.SaveChangesAsync();

        return CreatedAtAction("GetSeguceData", new { id = seguceData.Id }, seguceData);
    }

    // Autres actions CRUD
}
```

## Étape 9 : Tester l'API
Utilisez des outils comme Postman ou Swagger pour tester votre API et vous assurer que les données peuvent être stockées et récupérées correctement.

## Étape 10 : Déployer la solution
Déployez votre application ASP.NET Core sur un serveur web ou une plateforme cloud (comme Azure) selon vos besoins.

Avec ces étapes, vous devriez être en mesure de créer une base de données capable de stocker des données SEGUCE et DGDA, et de gérer les mouvements d'entrée OCC en utilisant ASP.NET Core.
