# Data Integration and API Exposure using ASP.NET Core

## Création d'une Base de Données avec ASP.NET Core pour SEGUCE et DGDA

Pour mettre en place une base de données capable de stocker des données pour SEGUCE et DGDA, y compris les données XML et Excel, ainsi que les mouvements d'entrée OCC, suivez les étapes détaillées ci-dessous :

## Étape 1 : Créer un Nouveau Projet ASP.NET Core

1. **Ouvrez Visual Studio.**
2. **Créez un nouveau projet ASP.NET Core Web Application.**
   - Lancez Visual Studio et sélectionnez `Créer un nouveau projet`.
   - Recherchez `ASP.NET Core Web Application`, sélectionnez-le, puis cliquez sur `Suivant`.
   - Nommez votre projet et choisissez l'emplacement de sauvegarde, puis cliquez sur `Créer`.
3. **Choisissez un modèle "API" ou "MVC" selon vos besoins.**
   - Dans la fenêtre `Créer une nouvelle application web ASP.NET Core`, sélectionnez soit le modèle `API`, soit `MVC` en fonction de l'architecture de votre application.

## Étape 2 : Configurer Entity Framework Core

1. **Installez les packages NuGet nécessaires pour Entity Framework Core.**
   - Dans Visual Studio, ouvrez le gestionnaire de package NuGet en cliquant avec le bouton droit sur votre projet dans l'explorateur de solutions, puis sélectionnez `Gérer les packages NuGet`.
   - Recherchez et installez les packages suivants :
     - `Microsoft.EntityFrameworkCore` : Le framework ORM pour accéder à la base de données.
     - `Microsoft.EntityFrameworkCore.SqlServer` : Le provider Entity Framework Core pour SQL Server.
     - `Microsoft.EntityFrameworkCore.Tools` : Outils utiles pour le développement avec EF Core.

Après avoir configuré votre projet ASP.NET Core et installé les packages nécessaires pour Entity Framework Core, vous êtes prêt à commencer le développement de votre application pour gérer et stocker des données SEGUCE et DGDA, y compris le traitement des fichiers XML et Excel ainsi que la gestion des mouvements d'entrée OCC.
## Project A: Service

### 1. Data Models

Define models to represent the data extracted from XML and stored in the database.

```csharp
public class Announcement
{
    public int Id { get; set; }
    public string AnnouncementID { get; set; }
    public string Sender { get; set; }
    public string Receiver { get; set; }
    public DateTime Date { get; set; }
    public string Reference { get; set; }
    public int AnnouncementType { get; set; }
    public string Function { get; set; }
    public string State { get; set; }
    public string Announcer { get; set; }
    public string Place { get; set; }
    public List<Transport> Transports { get; set; }
    public List<HandlingUnit> HandlingUnits { get; set; }
    public List<AnnouncementDocument> AnnouncementDocuments { get; set; }
    public List<AdditionalInformation> AdditionalInformations { get; set; }
}
```

### 2. Database Context

```csharp
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }

    public DbSet<Announcement> Announcements { get; set; }
    // Define other DbSet properties
}
```

### 3. Data Processing Service

```csharp
public class DataIntegrationService : BackgroundService
{
    private readonly ILogger<DataIntegrationService> _logger;
    private readonly ApplicationDbContext _context;
    private readonly IConfiguration _configuration;

    public DataIntegrationService(ILogger<DataIntegrationService> logger, ApplicationDbContext context, IConfiguration configuration)
    {
        _logger = logger;
        _context = context;
        _configuration = configuration;
    }

    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            _logger.LogInformation("Data Integration Service running at: {time}", DateTimeOffset.Now);

            await ProcessXmlData();

            await Task.Delay(60000, stoppingToken); // Delay for 1 minute
        }
    }

    private async Task ProcessXmlData()
    {
        var xmlDirectory = _configuration.GetValue<string>("DataDirectories:Xml");
        foreach (var file in Directory.GetFiles(xmlDirectory, "*.xml"))
        {
            var xmlData = await File.ReadAllTextAsync(file);
            var announcement = DeserializeXml<Announcement>(xmlData);
            _context.Announcements.Add(announcement);
        }
        await _context.SaveChangesAsync();
    }

    private T DeserializeXml<T>(string xml)
    {
        var serializer = new XmlSerializer(typeof(T));
        using (var reader = new StringReader(xml))
        {
            return (T)serializer.Deserialize(reader);
        }
    }
}
```

### 4. Service Configuration

```csharp
public class Program
{
    public static void Main(string[] args)
    {
        CreateHostBuilder(args).Build().Run();
    }

    public static IHostBuilder CreateHostBuilder(string[] args) =>
        Host.CreateDefaultBuilder(args)
            .ConfigureServices((hostContext, services) =>
            {
                services.AddDbContext<ApplicationDbContext>(options =>
                    options.UseSqlServer(hostContext.Configuration.GetConnectionString("DefaultConnection")));
                services.AddHostedService<DataIntegrationService>();
            });
}
```

## Project B: AP

### 1. Service and Middleware Configuration

```csharp
public class Startup
{
    public IConfiguration Configuration { get; }

    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public void ConfigureServices(IServiceCollection services)
    {
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));
        
        services.AddControllers()
            .AddNewtonsoftJson(options =>
                options.SerializerSettings.ReferenceLoopHandling = Newtonsoft.Json.ReferenceLoopHandling.Ignore);
    }

    public void Configure(IApplicationBuilder app, IWebHostEnvironment env)
    {
        if (env.IsDevelopment())
        {
            app.UseDeveloperExceptionPage();
        }

        app.UseRouting();

        app.UseAuthorization();

        app.UseEndpoints(endpoints =>
        {
            endpoints.MapControllers();
        });
    }
}
```

### 2. API Controllers

```csharp
[ApiController]
[Route("api/[controller]")]
public class AnnouncementController : ControllerBase
{
    private readonly ApplicationDbContext _context;

    public AnnouncementController(ApplicationDbContext context)
    {
        _context = context;
    }

    [HttpGet]
    public async Task<ActionResult<IEnumerable<Announcement>>> GetAnnouncements()
    {
        return await _context.Announcements.ToListAsync();
    }

    [HttpGet("{id}")]
    public async Task<ActionResult<Announcement>> GetAnnouncement(int id)
    {
        var announcement = await _context.Announcements.FindAsync(id);
        if (announcement == null)
        {
            return NotFound();
        }

        return announcement;
    }

    [HttpPost]
    public async Task<ActionResult<Announcement>> PostAnnouncement(Announcement announcement)
    {
        _context.Announcements.Add(announcement);
        await _context.SaveChangesAsync();

        return CreatedAtAction("GetAnnouncement", new { id = announcement.Id }, announcement);
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> PutAnnouncement(int id, Announcement announcement)
    {
        if (id != announcement.Id)
        {
            return BadRequest();
        }

        _context.Entry(announcement).State = EntityState.Modified;

        try
        {
            await _context.SaveChangesAsync();
        }
        catch (DbUpdateConcurrencyException)
        {
            if (!AnnouncementExists(id))
            {
                return NotFound();
            }
            else
            {
                throw;
            }
        }

        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteAnnouncement(int id)
    {
        var announcement = await _context.Announcements.FindAsync(id);
        if (announcement == null)
        {
            return NotFound();
        }

        _context.Announcements.Remove(announcement);
        await _context.SaveChangesAsync();

        return NoContent();
    }

    private bool AnnouncementExists(int id)
    {
        return _context.Announcements.Any(e => e.Id == id);
    }
}
```

## Conclusion

This setup outlines the integration of XML data into a local database and its exposure through a RESTful API using ASP.NET Core. The data integration service processes XML files, deserializes them, and stores the data in the database. The API exposes this data to other stakeholders.

# Configuration Options for ASP.NET Core Application

## ConnectionStrings
The `ConnectionStrings` section contains settings used to establish connections to the database.

### DefaultConnection
This is the name of the connection string. It is a key that the application uses to access this specific connection string settings.

- **Server=(localdb)\\mssqllocaldb**: Points to a SQL Server instance; in this case, it's using LocalDB, which is a lightweight version of SQL Server Express designed for app development.
- **Database=DataIntegrationDb**: Specifies the name of the database to connect to.
- **Trusted_Connection=True**: Uses Windows Authentication to access SQL Server. This means the user's Windows credentials are used for authentication.
- **MultipleActiveResultSets=true**: Enables the use of multiple active result sets in a single connection. This feature allows executing multiple batches on a single connection.

## DataDirectories
The `DataDirectories` section is used to define paths to directories that the application will use to read or write files.

### Xml
This is a custom key representing the directory where XML files are stored. The application reads XML files from this location to process and load into the database.

```json
"DataDirectories": {
  "Xml": "C:\\Path\\To\\XmlFiles"
}
```
# Logging

The Logging section specifies how the application logs runtime events. ASP.NET Core uses `Microsoft.Extensions.Logging` to log various events, and settings here define the behavior of these logs.

## LogLevel

Controls the level of logging that is enabled.

- **Default**: "Information" level logs will log informational messages, warnings, errors, and critical errors. This is the default logging level for unspecified categories.
- **Microsoft**: Only logs warnings and above for all Microsoft namespaces. This is useful to reduce verbosity from framework logs.
- **Microsoft.Hosting.Lifetime**: Controls the log level specifically for log messages that relate to the app's lifetime (e.g., starting, stopping).

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning",
      "Microsoft.Hosting.Lifetime": "Information"
    }
  }
}
```
# Applying the Configuration in the Application

These configurations are typically accessed in the application through the `IConfiguration` interface, which is injected into services or options where needed.

```csharp
public class Startup
{
    public IConfiguration Configuration { get; }

    public Startup(IConfiguration configuration)
    {
        Configuration = configuration;
    }

    public void ConfigureServices(IServiceCollection services)
    {
        // Accessing connection string
        services.AddDbContext<ApplicationDbContext>(options =>
            options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));
        
        // Accessing XML directory path
        var xmlDirectory = Configuration["DataDirectories:Xml"];
    }
}
```

This setup allows your application to be modular and configurable, with settings that can be changed without modifying the code, making it easier to adapt to different environments like development, testing, and production.

# Projet A: Service de Traitement des Données
Ce projet est chargé de la lecture, du traitement et de l'intégration des données XML dans une base de données locale. Voici les grandes lignes de son fonctionnement :

- **Lecture des fichiers XML :** Le service surveille un répertoire spécifié pour détecter de nouveaux fichiers XML. Ce chemin peut être configuré dans le fichier `appsettings.json` sous `DataDirectories:Xml`.

- **Désérialisation des données XML :** Lorsque de nouveaux fichiers sont détectés, ils sont lus et les données XML sont désérialisées en objets C# à l'aide de la classe `XmlSerializer`, qui convertit les éléments XML en objets C# basés sur des modèles prédéfinis.

- **Stockage des données :** Après la conversion des données en objets, elles sont stockées dans une base de données SQL Server locale. Les informations de connexion à cette base sont configurées dans `ConnectionStrings` dans `appsettings.json`.

- **Traitement continu :** Le service fonctionne en continu, vérifiant périodiquement la présence de nouveaux fichiers XML à traiter. Cette vérification est configurée pour s'exécuter à des intervalles réguliers, par exemple, toutes les minutes.

# Projet B: API pour Exposer les Données
Une fois les données stockées dans la base de données locale, le Projet B les rend accessibles via une API RESTful. Voici les principales caractéristiques de ce projet :

- **Endpoints API :** Des endpoints tels que GET, POST, PUT et DELETE sont mis en place pour permettre aux utilisateurs ou aux systèmes clients de récupérer, ajouter, modifier ou supprimer des données. Chaque endpoint correspond à une opération spécifique sur les données.

- **Sérialisation JSON :** Lorsqu'une requête est faite à l'API, les données récupérées de la base de données sont sérialisées en JSON avant d'être renvoyées au client, facilitant ainsi l'intégration avec d'autres systèmes ou applications frontales.

- **Gestion des erreurs :** L'API gère les erreurs telles que les entrées non trouvées ou les erreurs de serveur interne, en renvoyant des réponses appropriées, y compris des codes d'état HTTP et des messages d'erreur clairs.

- **Sécurité et Autorisation :** Pour une application en production, il est possible d'ajouter des couches de sécurité, comme l'authentification et l'autorisation, pour contrôler l'accès aux données exposées par l'API.

## Intégration et Flux de Données
Le flux de données commence avec l'importation de fichiers XML dans le système (Projet A), où ils sont traités et stockés dans la base de données. Ces données peuvent ensuite être consultées et manipulées via l'API RESTful (Projet B), permettant une interaction externe avec les données.

## Conclusion
Cette configuration assure une séparation claire des responsabilités : le Projet A s'occupe du traitement interne des données, tandis que le Projet B agit comme interface pour les interactions externes avec ces données. Cette architecture organise de manière efficace le flux de travail, rendant le système à la fois évolutif et facile à maintenir.

# Gestion du Stockage des Données et du Traitement Continu dans une Application ASP.NET Core

Pour mieux comprendre le fonctionnement interne de l'application, notamment en ce qui concerne le stockage des données et le traitement continu, examinons des extraits de code spécifiques à chaque fonctionnalité. L'application utilise ASP.NET Core avec Entity Framework Core pour la gestion de la base de données.

## Stockage des Données

Le stockage des données dans une base de données SQL Server locale est réalisé via Entity Framework Core. La configuration et l'utilisation de ce framework sont détaillées ci-dessous :

### Configuration de la Base de Données

Dans `Startup.cs` ou `Program.cs`, la base de données est configurée comme suit :

```csharp
services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlServer(Configuration.GetConnectionString("DefaultConnection")));
```

- `AddDbContext<ApplicationDbContext>` : Ajoute le contexte de la base de données comme service dans l'application. `ApplicationDbContext` est une classe qui étend `DbContext` d'Entity Framework Core et configure les modèles de données et leurs relations.
- `UseSqlServer` : Configure le contexte pour utiliser SQL Server avec la chaîne de connexion fournie.

#### Extrait de `appsettings.json`

```json
"ConnectionStrings": {
  "DefaultConnection": "Server=(localdb)\\mssqllocaldb;Database=DataIntegrationDb;Trusted_Connection=True;MultipleActiveResultSets=true"
}
```

Cette chaîne de connexion, `DefaultConnection`, est utilisée pour connecter l'application à SQL Server, spécifiant le serveur de base de données (LocalDB ici), le nom de la base de données, et d'autres paramètres comme l'utilisation de l'authentification Windows et l'activation de plusieurs ensembles de résultats actifs simultanément.

### Sauvegarde des Données

```csharp
var announcement = DeserializeXml<Announcement>(xmlData);
_context.Announcements.Add(announcement);
await _context.SaveChangesAsync();
```

- `DeserializeXml<Announcement>` : Convertit les données XML en objet `Announcement`.
- `_context.Announcements.Add(announcement)` : Ajoute l'objet `Announcement` au contexte de la base de données.
- `_context.SaveChangesAsync()` : Enregistre toutes les modifications apportées dans le contexte de la base de données à la base de données SQL Server, réalisant ainsi le stockage des données.

## Traitement Continu

Le traitement continu est assuré par un service en arrière-plan dans ASP.NET Core, qui vérifie régulièrement les nouveaux fichiers XML pour les traiter.

### Configuration du Service en Arrière-Plan

Dans `Program.cs`, le service en arrière-plan est configuré comme suit :

```csharp
services.AddHostedService<DataIntegrationService>();
```

- `AddHostedService<DataIntegrationService>` : Enregistre le service `DataIntegrationService` comme un service en arrière-plan qui démarre avec l'application et s'exécute jusqu'à ce que l'application s'arrête.

### Implémentation du Service en Arrière-Plan

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    while (!stoppingToken.IsCancellationRequested)
    {
        _logger.LogInformation("Data Integration Service running at: {time}", DateTimeOffset.Now);
        await ProcessXmlData();
        await Task.Delay(60000, stoppingToken); // Delay for 1 minute
    }
}
```

- `ExecuteAsync` : Cette méthode est appelée lorsque le service en arrière-plan démarre. Elle exécute une boucle qui continue tant que l'application s'exécute.
- `ProcessXmlData` : Méthode qui traite les fichiers XML.
- `Task.Delay` : Attend une minute avant de vérifier à nouveau les fichiers, permettant au service de ne pas monopoliser les ressources et de vérifier les nouveaux fichiers périodiquement.

## Conclusion

Ces extraits illustrent comment les données sont stockées de manière persistante dans une base de données et comment un service en arrière-plan peut traiter les données de manière continue, en vérifiant les nouveaux fichiers XML à intervalles réguliers. Ce modèle est typique pour les applications de traitement de données en lot et les systèmes d'intégration continue.