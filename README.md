# Offset vs cursor pagination

Companion code for the video [Paginação Eficiente: Offset vs Cursor pagination](https://youtu.be/97sCrxvEgsY) (in Portuguese). The same list endpoint written both ways, with a seeder that grows the table into the millions so the difference shows up.

## The problem

`OFFSET` pagination asks the database to walk past every row before the page and throw it away. Page 1 is cheap. Page 50,000 reads a million rows to return ten. The cost grows with the page number, and users scrolling a feed hit exactly those deep pages.

## The decision

Cursor (keyset) pagination. Instead of "give me page N", the client sends the last id it saw and the query becomes a range scan on the index:

```csharp
// offset: skip (page - 1) * size rows, then take size
db.Products.OrderByDescending(x => x.Id).Skip((currentPage - 1) * perPage).Take(perPage)

// cursor: start right after the last row the client saw
db.Products.OrderByDescending(x => x.Id).Where(x => x.Id < lastSeen).Take(perPage)
```

Every page costs the same, whatever its position. The API returns the cursor in a `last-seen` response header, and the client passes it back as `?lastSeen=`.

## The trade-off

- No "jump to page 7" and no total page count. Cursor pagination is for feeds, infinite scroll and API consumers, not for a grid with numbered pages.
- The sort key must be unique and stable. Here it's the identity `Id`; sorting by `CreatedAt` would need a tiebreaker.
- Inserts and deletes between requests don't shift the window, which is a feature for feeds (no duplicated or skipped items) and a surprise for anyone expecting page numbers to be stable.

## Endpoints

| | Offset | Cursor |
|---|---|---|
| Route | `GET /products` | `GET /v2/products` |
| Params | `currentPage`, `perPage` | `lastSeen`, `perPage` |
| Next page | `currentPage + 1` | value of the `last-seen` response header |

Swagger UI is on at the root while running.

## Run

```bash
docker compose up -d                 # SQL Server 2022
dotnet run --project Seeder          # inserts 20,000 products per second until you stop it
dotnet run --project Api
```

Let the seeder run for a few minutes, then compare `/products?currentPage=50000` with the equivalent depth through `/v2/products`.

## Projects

- `Data`: EF Core context and the `Product` model
- `Seeder`: background worker that batches fake products with Bogus
- `Api`: minimal API with both endpoints

.NET 8 · EF Core · SQL Server
