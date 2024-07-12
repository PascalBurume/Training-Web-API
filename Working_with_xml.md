# Data Integration and API Exposure using ASP.NET Core

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

# Applying the Configuration in the Application

These configurations are typically accessed in the application through the `IConfiguration` interface, which is injected into services or options where needed.

```charp
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
