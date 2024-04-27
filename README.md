## Diseño de una solución de seguiento de firmas de un documento
Seguimiento de firmas sobre un documento que pasa secuencialmente a un grupo de firmantes. Se define el modelo ER, procedimientos almacenados y los disparadores necesarios para manejar la secuencia y el estado de las firmas.

## Diagrama de secuencias de la solución 
![Diagrama DFLOW](https://github.com/jose-dev-gt/signature-tracking/blob/main/diagSeq.png)

## Modelo Entidad-Relación(ER)
Entidades y Atributos

1. Documento
* DocumentoID (PK): Identificador único del documento.
* Titulo: Título del documento.
* Contenido: Contenido del documento.
* FechaCreacion: Fecha de creación del documento.
2. Firmante
* FirmanteID (PK): Identificador único del firmante.
* Nombre: Nombre del firmante.
* Email: Email del firmante.
3. Firma
* FirmaID (PK): Identificador único de la firma.
* DocumentoID (FK): Identificador del documento a firmar.
* FirmanteID (FK): Identificador del firmante.
* FechaFirma: Fecha en la que el firmante firma el documento.
* Estado: Estado de la firma (Pendiente, Completada, Rechazada).
4. SecuenciaFirma
* SecuenciaID (PK): Identificador único de la secuencia.
* DocumentoID (FK): Identificador del documento asociado.
* FirmanteID (FK): Identificador del próximo firmante en la secuencia.
* Orden: Orden en el que el firmante debe firmar.
5. Relaciones
* Un Documento puede tener múltiples Firmas.
* Un Firmante puede tener múltiples Firmas.
* Un Documento tiene una SecuenciaFirma, que define el orden en que los firmantes deben firmar.

## Diagrama ER
![Diagrama ER](https://github.com/jose-dev-gt/signature-tracking/blob/main/diagERSig.png)

## Procedimientos almacenados
1. Crear un nuevo documento y establecer la secuencia inicial de firmas
```sql
CREATE PROCEDURE spCrearDocumento
    @Titulo NVARCHAR(255),
    @Contenido TEXT,
    @Firmantes TABLE(FirmanteID INT, Orden INT)
AS
BEGIN
    DECLARE @DocumentoID INT;
    INSERT INTO Documento (Titulo, Contenido, FechaCreacion)
    VALUES (@Titulo, @Contenido, GETDATE());

    SET @DocumentoID = SCOPE_IDENTITY();

    INSERT INTO SecuenciaFirma (DocumentoID, FirmanteID, Orden)
    SELECT @DocumentoID, FirmanteID, Orden FROM @Firmantes;
END;
GO

```
2. Firmar un documento si es su turno según la secuencia.
```sql
CREATE PROCEDURE spFirmarDocumento
    @DocumentoID INT,
    @FirmanteID INT,
    @Estado NVARCHAR(50)
AS
BEGIN
    DECLARE @OrdenActual INT;
    SELECT @OrdenActual = Orden FROM SecuenciaFirma
    WHERE DocumentoID = @DocumentoID AND FirmanteID = @FirmanteID;

    IF EXISTS(
        SELECT 1 FROM Firma
        WHERE DocumentoID = @DocumentoID AND Estado = 'Pendiente' AND Orden < @OrdenActual
    )
    BEGIN
        RAISERROR('No es su turno para firmar.', 16, 1);
    END
    ELSE
    BEGIN
        INSERT INTO Firma (DocumentoID, FirmanteID, FechaFirma, Estado)
        VALUES (@DocumentoID, @FirmanteID, GETDATE(), @Estado);
    END
END;
GO

```

## Disparadores
1. Actualizar estado de firma
```sql
CREATE TRIGGER trgActualizarEstadoFirma
ON Firma
AFTER UPDATE
AS
BEGIN
    IF UPDATE(Estado)
    BEGIN
        DECLARE @DocumentoID INT = inserted.DocumentoID;
        DECLARE @Estado NVARCHAR(50) = inserted.Estado;

        UPDATE Documento
        SET Estado = @Estado
        WHERE DocumentoID = @DocumentoID;
    END
END;
GO

```
## Flujo para hacer inserts o updates
Flujo de operaciones de inserción y actualización en un backend C#, podemos utilizar un enfoque típico que involucre capas de acceso a datos, lógica de negocio y presentación. EL flujo de datos simplificado que incluye operaciones CRUD (Create, Read, Update, Delete) centradas en las operaciones de insert y update:

![Diagrama API](https://github.com/jose-dev-gt/signature-tracking/blob/main/diagAPI.png)

## Componentes del flujo
1. API Controllers: Puntos de entrada para las operaciones HTTP.
2. Services: Clases que contienen la lógica de negocio.
3. Data Access Layer (DAL): Capa que maneja la comunicación directa con la base de datos.
4. Database: La base de datos SQL Server donde se guardan los datos.

## Modelo
```csharp
public class Item
{
    public int Id { get; set; }
    public string Name { get; set; }
    public string Description { get; set; }
}

```
## Controller
```csharp
[ApiController]
[Route("[controller]")]
public class ItemsController : ControllerBase
{
    private readonly IItemService _itemService;

    public ItemsController(IItemService itemService)
    {
        _itemService = itemService;
    }

    [HttpPost]
    public IActionResult CreateItem(Item item)
    {
        var result = _itemService.CreateItem(item);
        if (result)
            return Ok();
        else
            return BadRequest();
    }

    [HttpPut("{id}")]
    public IActionResult UpdateItem(int id, Item item)
    {
        var result = _itemService.UpdateItem(id, item);
        if (result)
            return Ok();
        else
            return BadRequest();
    }
}

```
## Service Layer
```csharp
public class ItemService : IItemService
{
    private readonly MyDbContext _context;

    public ItemService(MyDbContext context)
    {
        _context = context;
    }

    public bool CreateItem(Item item)
    {
        _context.Items.Add(item);
        return _context.SaveChanges() > 0;
    }

    public bool UpdateItem(int id, Item updatedItem)
    {
        var item = _context.Items.Find(id);
        if (item == null) return false;
        
        item.Name = updatedItem.Name;
        item.Description = updatedItem.Description;
        return _context.SaveChanges() > 0;
    }
}

```
