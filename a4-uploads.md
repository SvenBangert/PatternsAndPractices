# Uploads

[Table of Contents](./toc.md)

* [Overview](#overview)
* [.NET Core](#net-core)
    * [Configuration](#configuration)
    * [Data Layer](#data-layer)
    * [Business Logic](#business-logic)
    * [Controller](#controller)
* [Angular](#angular)
    * [Model](#model)
    * [Service](#service)
    * [Pipe](#pipe)
    * [Components](#components)
    * [Dialogs](#dialogs)
    * [Route](#route)
* [Related Data](#related-data)

## [Overview](#uploads)

The [File API](https://developer.mozilla.org/en-US/docs/Web/API/File/Using_files_from_web_applications) defines a mechanism for allowing users to upload binary file data. This article will discuss setting up .NET Core to manage these uploads, building a workflow in Angular, and tying uploads to related data via Entity Framework.

## [.NET Core](#uploads)

### [Configuration](#uploads)

Before building out entities or API endpoints, it's important to understand how uploaded files will be stored and accessed. Rather than storing files as binary data in the database, they will be stored at a location that can be referenced by a SQL record. This will keep the database from growing unnecessarily huge, and reduce the amount of time needed to retrieve files.

To make the location of the file storage configurable, an `UploadConfig` class is created that specifies a `DirectoryBasePath` as well as a `UrlBasePath`. The `DirectoryBasePath` refers to the physical location where files will be stored, and the `UrlBasePath` refers to the root URL the files can be reference from in the app.

**`UploadConfig.cs`**

```cs
namespace {Project}.Core.Upload
{
    public class UploadConfig
    {
        public string DirectoryBasePath { get; set; }
        public string UrlBasePath { get; set; }
    }
}
```

During development, files will be stored in the `{Project}.Web/wwwroot/files/` directory. In any other environment, the directory and URL paths will be specified via `AppDirectoryBasePath` and `AppUrlBasePath` environment variables. Here is the relevant `Startup.cs` configuration for registering an `UploadConfig` instance with the dependency injection container:

**`Startup.cs`**

```cs
public class Startup
{
    private void SetupDevelopmentDirectories(IHostingEnvironment env)
    {
        if (env.IsDevelopment())
        {
            if (!Directory.Exists($@"{env.WebRootPath}/files"))
            {
                Directory.CreateDirectory($@"{env.WebRootPath}/files");
            }
        }
    }

    public Startup(IConfiguration configuration, IHostingEnvironment environment)
    {
        SetupDevelopmentDirectories(environment);
        // additional constructor logic
    }

    public void ConfigureServices(IServiceCollection services)
    {
        // preceding service registrations

        if (Environment.IsDevelopment())
        {
            services.AddSingleton(new UploadConfig
            {
                DirectoryBasePath = $@"{Environment.WebRootPath}/files/",
                UrlBasePath = "/files/"
            });
        }
        else
        {
            services.AddSingleton(new UploadConfig
            {
                DirectoryBasePath = Configuration.GetValue<string>("AppDirectoryBasePath"),
                UrlBasePath = Configuration.GetValue<string>("AppUrlBasePath")
            });
        }

        // additional service registrations
    }
}
```  

The `SetupDevelopmentDirectories()` method that is defined, then called in the `Startup` constructor, ensures that the path specified when registering `UploadConfig` exists before it is registered.  

`UploadConfig` is then registered as a **Singleton** in `ConfigureServices`. If in the **Development** environment, the path will default to `{Project}.Web/wwwroot/files/`. Otherwise, the path will be retrieved from the `AppDirectoryBasePath` and `AppUrlBasePath` environment variables.

### [Data Layer](#uploads)

File uploads will be tracked in the database as an instance of an `Upload` entity, which is defined as follows:

**`Upload.cs`**

```cs
namespace UploadDemo.Data.Entities
{
    public class Upload
    {
        public int Id { get; set; }
        public string Url { get; set; }
        public string Path { get; set; }
        public string File { get; set; }
        public string Name { get; set; }
        public string FileType { get; set; }
        public long Size { get; set; }
        public DateTime UploadDate { get; set; }
        public bool IsDeleted { get; set; }
    }
}
```

Property | Description
---------|------------
`Url` | Specifies the URL the file can be served from in the application
`Path` | Specifies the fully qualified, physical location the file is located at
`File` | The original file name, including the file extension
`Name` | The URL-encoded name, including the file extension
`FileType` | The [MIME type](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/MIME_types)
`Size` | The size, in bytes
`UploadDate` | Date the file is uploaded
`IsDeleted` | A flag indicating whether or not an upload has been soft deleted

Add `Upload` to an `AppDbContext` class:

**`AppDbContext.cs`**

``` cs
namespace UploadDemo.Data
{
    public class AppDbContext : DbContext
    {
        public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }
        
        public DbSet<Upload> Uploads { get; set; }

        protected override void OnModelCreating(ModelBuilder modelBuilder)
        {
            modelBuilder
                .Model
                .GetEntityTypes()
                .ToList()
                .ForEach(x =>
                {
                    modelBuilder
                        .Entity(x.Name)
                        .ToTable(x.Name.Split('.').Last());
                });
        }
    }
}
```

Register `AppDbContext` with the `Startup` class:

**`Startup.cs`**

``` cs
public void ConfigureServices(IServiceCollection services)
{
    services.AddDbContext<AppDbContext>(options => options.UseSqlServer(Configuration.GetConnectionString("Default"));
}
```

> Make sure to generate a migration and update the database as outlined in the [Data Access Layer - Database Management Workflow](./02-data-access-layer.md#database-management-workflow) article

### [Business Logic](#uploads)  

Getting uploads works the same as any other retrieval methods that have been shown at this point. For completeness, they will be shown with the rest of the methods. It is the create and delete methods that require some extra work.

**`UploadExtensions.cs`**

```cs
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
using Microsoft.AspNetCore.Http;
using Microsoft.EntityFrameworkCore;
using UploadDemo.Core.Extensions;
using UploadDemo.Data.Entities;

namespace UploadDemo.Data.Extensions
{
    public static class UploadExtensions
    {
        public static async Task<List<Upload>> GetUploads(this AppDbContext db, bool isDeleted = false)
        {
            var uploads = await db.Uploads
                .Where(x => x.IsDeleted == isDeleted)
                .OrderByDescending(x => x.UploadDate)
                .ToListAsync();

            return uploads;
        }

        public static async Task<List<Upload>> SearchUploads(this AppDbContext db, string search, bool isDeleted = false)
        {
            search = search.ToLower();
            var uploads = await db.Uploads
                .Where(x => x.IsDeleted == isDeleted)
                .Where(x => x.File.ToLower().Contains(search))
                .OrderByDescending(x => x.UploadDate)
                .ToListAsync();

            return uploads;
        }

        public static async Task<Upload> GetUpload(this AppDbContext db, int uploadId) => 
            await db.Uploads
                .SetUploadIncludes()
                .FirstOrDefaultAsync(x => x.Id == uploadId);

        public static async Task<Upload> GetUploadByName(this AppDbContext db, string file) => 
            await db.Uploads
                .SetUploadIncludes()
                .FirstOrDefaultAsync(x => x.File.ToLower() == file.ToLower());

        public static async Task<List<Upload>> UploadFiles(this AppDbContext db, IFormFileCollection files, string path, string url)
        {
            if (files.Count < 1)
            {
                throw new Exception("No files provided for upload");
            }

            List<Upload> uploads = new List<Upload>();

            foreach (var file in files)
            {
                uploads.Add(await db.AddUpload(file, path, url));
            }

            return uploads;
        }

        public static async Task ToggleUploadDeleted(this AppDbContext db, Upload upload)
        {
            db.Uploads.Attach(upload);
            upload.IsDeleted = !upload.IsDeleted;
            await db.SaveChangesAsync();
        }

        public static async Task RemoveUpload(this AppDbContext db, Upload upload)
        {
            await upload.DeleteFile();
            db.Uploads.Remove(upload);
            await db.SaveChangesAsync();
        }

        static async Task<Upload> AddUpload(this AppDbContext db, IFormFile file, string path, string url)
        {
            var upload = await file.WriteFile(path, url);
            upload.UploadDate = DateTime.Now;
            await db.Uploads.AddAsync(upload);
            await db.SaveChangesAsync();
            return upload;
        }

        static async Task<Upload> WriteFile(this IFormFile file, string path, string url)
        {
            if (!(Directory.Exists(path)))
            {
                Directory.CreateDirectory(path);
            }

            var upload = await file.CreateUpload(path, url);

            using (var stream = new FileStream(upload.Path, FileMode.Create))
            {
                await file.CopyToAsync(stream);
            }

            return upload;
        }

        static Task<Upload> CreateUpload(this IFormFile file, string path, string url) => Task.Run(() =>
        {
            var f = file.CreateSafeName(path);

            var upload = new Upload
            {
                File = f,
                Name = file.Name,
                Path = $"{path}{f}",
                Url = $"{url}{f}",
                FileType = file.ContentType,
                Size = file.Length
            };

            return upload;
        });

        static string CreateSafeName(this IFormFile file, string path)
        {
            var increment = 0;
            var fileName = file.FileName.UrlEncode();
            var newName = fileName;

            while (File.Exists(path + newName))
            {
                var extension = fileName.Split('.').Last();
                newName = $"{fileName.Replace($".{extension}", "")}_{++increment}.{extension}";
            }

            return newName;
        }

        static Task DeleteFile(this Upload upload) => Task.Run(() =>
        {
            try
            {
                if (File.Exists(upload.Path))
                {
                    File.Delete(upload.Path);
                }
            }
            catch (Exception ex)
            {
                throw new Exception(ex.GetExceptionChain());
            }
        });
    }
}
```  

> The following will only describe the `UploadFiles` and `RemoveUpload` methods, as they pertain to the additional functionality inherent in file uploads. If the remaining methods are unfamiliar, refer to the [Business Logic](./03-business-logic.md) article.

We'll start with the `RemoveUpload` method, as it is substantially less complicated. 

* Call the private `DeleteFile` extension method
    * If the file specified at `Upload.Path` exists, delete the physical file
* Remove the associated `Upload` record from the database
* Save changes to the database

The `UploadFiles` extension method triggers a chain of defined method calls, each with a specific purpose. The following section will start with the deepest method, and work back up to `UploadFiles` so that you can see how the entire interaction occurs.

Method | Description
-------|------------
`CreateSafeName` | Extends on `IFormFile file` and accepts a `string path` argument. If the name of the provided file already exists at the specified path, a safe name is generated by adding an incremented number to the end of the name until a matching file is not found
`CreateUpload` | Extends on `IFormFile file` and accepts `string path` and `string url` arguments. Generates a safe name by calling the above `CreateSafeName` method. Generates an instance of the `Upload` class based on the safe name and properties of `file`, as well as the provided `path` and `url` arguments. The `Upload` instance is then returned.
`WriteFile` | Extends on `IFormFile file` and accepts `string path` and `string url` arguments. Ensures that the directory specified by the `path` argument exists. Then calls the above `CreateUpload` method to generate an `Upload` instance. The file is then written to the `path` specified using the `CopyToAsync` method of `IFormFile`, and the `Upload` instance is returned.
`AddUpload` | Extends on `AppDbContext db` and accepts `IFormFile file`, `string path`, and `string url` arguments. Calls the above `WriteFile` method to generate an instance of `Upload` and write the file contained by `file` to the specified `path`. The `UploadDate` property is set to `DateTime.Now`, and the `Upload` instance is added to the database. The changes are saved, and the `Upload` instance is returned.

The public `UploadFiles` method extends on `AppDbContext db` and accepts `IFormFileCollection files`, `string path`, and `string url` as arguments. If the amount of files provided to the method is less than one, an exception is thrown indicating that no files were provided.

Each file in the `IFormFileCollection` is iterated through and added to a `List<Upload> uploads` collection by calling the `AddUpload` method specified above. The resulting list of uploads is then returned to the caller.

### [Controller](#uploads)

Here is the controller that maps to the public methods specified for the `Upload` entity:

**`UploadController.cs`**

```cs
namespace UploadDemo.Web.Controllers
{
    [Route("api/[controller]")]
    public class UploadController : Controller
    {
        private AppDbContext db;
        private UploadConfig config;

        public UploadController(AppDbContext db, UploadConfig config)
        {
            this.db = db;
            this.config = config;
        }

        [HttpGet("[action]")]
        public async Task<List<Upload>> GetUploads() => await db.GetUploads();

        [HttpGet("[action]")]
        public async Task<List<Upload>> GetDeletedUploads() => await db.GetUploads(true);

        [HttpGet("[action]/{search}")]
        public async Task<List<Upload>> SearchUploads([FromRoute]string search) => await db.SearchUploads(search);

        [HttpGet("[action]/{search}")]
        public async Task<List<Upload>> SearchDeletedUploads([FromRoute]string search) => await db.SearchUploads(search, true);

        [HttpGet("[action]/{id}")]
        public async Task<Upload> GetUpload([FromRoute]int id) => await db.GetUpload(id);

        [HttpGet("[action]/{file}")]
        public async Task<Upload> GetUploadByName([FromRoute]string file) => await db.GetUploadByName(file);

        [HttpPost("[action]")]
        [DisableRequestSizeLimit]
        public async Task<List<Upload>> UploadFiles() =>
            await db.UploadFiles(
                Request.Form.Files,
                config.DirectoryBasePath,
                config.UrlBasePath
            );

        [HttpPost("[action]")]
        public async Task ToggleUploadDeleted([FromBody]Upload upload) => await db.ToggleUploadDeleted(upload);

        [HttpPost("[action]")]
        public async Task RemoveUpload([FromBody]Upload upload) => await db.RemoveUpload(upload);
    }
}
```

> With the exception of `UploadFiles`, this should all look familiar. If not, refer to the [Web API](./06-web-api.md) article.

By default, <span>ASP.NET</span> Core enforces a [**30MB** request size limit](https://github.com/aspnet/Announcements/issues/267) for HTTP requests. You can modify this default by either specifying a value using the [RequestSizeLimit](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.requestsizelimitattribute?view=aspnetcore-2.2) attribute on an endpoint, or bypassing limits completely by specifying the [DisableRequestSizeLimit](https://docs.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.mvc.disablerequestsizelimitattribute?view=aspnetcore-2.2) attribute on an endpoint. In this case, `UploadFiles` bypasses the limit completely by specifying `DisableRequestSize`.

The `UploadFiles` API endpoint maps to the `UploadFiles` method defined above. `IFormFileCollection` is extracted from the body of the request at `Request.Form.Files` as the first argument. The `DirectoryBasePath` and `UrlBasePath` properties of the injected `UploadConfig` object are passed as the second and third arguments respectively.

## [Angular](#uploads)

Now that .NET Core is setup to handle file uploads on the back-end, it's time to see how to bring it all together by actually provided file uploads from Angular.

### [Model](#uploads)  

An `Upload` TypeScript class is added that defines the shape of the `Upload` data defined by the back-end.

**`Upload.ts`**

```ts
export class Upload {
  id: number;
  url: string;
  path: string;
  file: string;
  name: string;
  fileType: string;
  size: number;
  uploadDate: Date;
  isDeleted: boolean;
}
```

### [Service](#uploads)

Before defining the `UploadService`, it's important to note that the `CoreService` defined by the template contains a `getUploadOptions()` function:

**`core.service.ts` - `getUploadOptions()`**  

```ts
getUploadOptions = (): HttpHeaders => {
  const headers = new HttpHeaders();
  headers.set('Accept', 'application/json');
  headers.delete('Content-Type');
  return headers;
}
```  

This appropriately conditions the headers for a `POST` request to send `FormData` to an API endpoint in .NET Core.

Using the standard convention of mapping API endpoints to an Angular service, an `UploadService` is defined:

**`upload.service.ts`**

```ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Subject } from 'rxjs';
import { CoreService } from './core.service';
import { SnackerService } from './snacker.service';
import { Upload } from '../models';


@Injectable()
export class UploadService {
  private uploads = new Subject<Upload[]>();
  private upload = new Subject<Upload>();

  uploads$ = this.uploads.asObservable();
  upload$ = this.upload.asObservable();

  constructor(
    private core: CoreService,
    private http: HttpClient,
    private snacker: SnackerService
  ) { }

  getUploads = () => this.http.get<Upload[]>('/api/upload/getUploads')
    .subscribe(
      data => this.uploads.next(data),
      err => this.snacker.sendErrorMessage(err.error)
    );

  getDeletedUploads = () => this.http.get<Upload[]>('/api/upload/getDeletedUploads')
    .subscribe(
      data => this.uploads.next(data),
      err => this.snacker.sendErrorMessage(err.error)
    );

  searchUploads = (search: string) => this.http.get<Upload[]>(`/api/upload/searchUploads/${search}`)
    .subscribe(
      data => this.uploads.next(data),
      err => this.snacker.sendErrorMessage(err.error)
    );

  searchDeletedUploads = (search: string) => this.http.get<Upload[]>(`/api/upload/getDeletedUploads/${search}`)
    .subscribe(
      data => this.uploads.next(data),
      err => this.snacker.sendErrorMessage(err.error)
    );

  getUpload = (id: number): Promise<boolean> =>
    new Promise((resolve) => {
      this.http.get<Upload>(`/api/upload/getUpload/${id}`)
      .subscribe(
        data => {
          this.upload.next(data);
          resolve(true);
        },
        err => {
          this.snacker.sendErrorMessage(err.error);
          resolve(false);
        }
      )
    });

  getUploadByName = (file: string): Promise<boolean> =>
    new Promise((resolve) => {
      this.http.get<Upload>(`/api/upload/getUploadByName/${file}`)
        .subscribe(
        data => {
          this.upload.next(data);
          resolve(true);
        },
        err => {
          this.snacker.sendErrorMessage(err.error);
          resolve(false);
        }
      )
    });

  uploadFiles = (formData: FormData): Promise<Upload[]> =>
    new Promise((resolve) => {
      this.http.post<Upload[]>('/api/upload/uploadFiles', formData, { headers: this.core.getUploadOptions() })
        .subscribe(
          res => {
            this.snacker.sendSuccessMessage('Uploads successfully processed');
            resolve(res);
          },
          err => {
            this.snacker.sendErrorMessage(err.error);
            resolve(null);
          }
        )
    });

  toggleUploadDeleted = (upload: Upload): Promise<boolean> =>
    new Promise((resolve) => {
      this.http.post('/api/upload/toggleUploadDeleted', upload)
        .subscribe(
          () => {
            const message = upload.isDeleted ?
              `${upload.file} successfully restored` :
              `${upload.file} successfully deleted`;

            this.snacker.sendSuccessMessage(message);
            resolve(true);
          },
          err => {
            this.snacker.sendErrorMessage(err.error);
            resolve(false);
          }
        )
    });

  removeUpload = (upload: Upload): Promise<boolean> =>
    new Promise((resolve) => {
      this.http.post('/api/upload/removeUpload', upload)
        .subscribe(
          () => {
            this.snacker.sendSuccessMessage(`${upload.file} permanently deleted`);
            resolve(true);
          },
          err => {
            this.snacker.sendErrorMessage(err.error);
            resolve(false);
          }
        )
    });
}
```  

> If you're unfamiliar with Angular Services, see the [Services](./13-services.md) article.  

The `uploadFiles` function receives a `formData: FormData` argument and returns a `Promise<Upload[]>`. An `HttpClient.post` function is called, providing `formData` in the body of the request, and setting the request headers by calling `CoreService.getUploadOptions()`. When the result returns, the user is alerted, and the results are resolved in the `Promise<Upload[]>` returned by the function. If an error occurs, the error is alerted to the user and `null` is resolved by the returned Promise.

### [Pipe](#uploads)

The `size` property of the `Upload` class is provided in bytes. In order to provide better labeling for file sizes, a `BytesPipe` is created:

**`bytes.pipe.ts`**

```ts
import {
  Pipe,
  PipeTransform
} from '@angular/core';

@Pipe({
  name: 'bytes'
})
export class BytesPipe implements PipeTransform {
  transform(value: number, precision = 2) {
    if (!value || value === 0) return '0 Bytes';
    const k = 1024,
          dm = precision <= 0 ? 0 : precision || 2,
          sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB'],
          i = Math.floor(Math.log(value) / Math.log(k));

    return parseFloat((value / Math.pow(k, i)).toFixed(dm)) + ' ' + sizes[i];
  }
}
```

The `transform` function takes the provided value, and uses the floor of the logarithmic relationship between `k` (**1024**) and `value` to determine its logarithmic base. This exponential relationship of `k` and the logarithmic base is then divided by `value` to determine the natural size, along with an identifier for the size of the value represented, to the precision specified by the `precision` argument (defaults to **2**), and is returned.

> Math documentation:
> * [Math.floor](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/floor)
> * [Math.log](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/log)
> * [Math.pow](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Math/pow)

Here's an example that demonstrates each step in pure JavaScript:

```js
const value = 23949380;
const k = 1024;
const sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB'];
const i = Math.floor(Math.log(k) / Math.log(value));
console.log('base', i); // "base" 2

const result = parseFloat((value / Math.pow(k, i)).toFixed(2)) + ' ' + sizes[i];
console.log('result', result); // "result" "22.84 MB"
```

### [Components](#uploads)

> To re-iterate the usefulness of the `FileUploadComponent` demonstrated in the [Display Components - FileUploadComponent](./16-display-components.md#fileuploadcomponent) article, it is defined here again so you can see how it is used in the context of file uploads. Refer to that section of the article for a detailed description of the component.

**`file-upload.component.css`**

```css
input[type=file] {
  display: none;
}
```

**`file-upload.component.html`**

```html
<input type="file"
       (change)="fileChange($event)"
       #fileInput
       [accept]="accept"
       [multiple]="multiple">
<button mat-button
        [color]="color"
        (click)="fileInput.click()">{{label}}</button>
```

**`file-upload.component.ts`**

```ts
import {
  Component,
  EventEmitter,
  Input,
  Output,
  ViewChild,
  ElementRef
} from '@angular/core';

@Component({
  selector: 'file-upload',
  templateUrl: 'file-upload.component.html',
  styleUrls: ['file-upload.component.css']
})
export class FileUploadComponent {
  @ViewChild('fileInput') fileInput: ElementRef;
  @Input() accept = '*/*';
  @Input() color = 'primary';
  @Input() label = 'Browse...';
  @Input() multiple = true;
  @Output() selected = new EventEmitter<[File[], FormData]>();

  fileChange = (event: any) => {
    const files: FileList = event.target.files;
    const fileList = new Array<File>();
    const formData = new FormData();

    for (let i = 0; i < files.length; i++) {
      formData.append(files.item(i).name, files.item(i));
      fileList.push(files.item(i));
    }

    this.selected.emit([fileList, formData]);
    this.fileInput.nativeElement.value = null;
  }
}
```  

The most important part of the `FileUploadComponent` in this context is that it allows users to select files, and returns these selected files as both an `Array<File>` for display purposes, and `FormData` with the files appended to it for passing to the `UploadService.uploadFiles` function.

A `FileListComponent` is created for being able to consistently render an `Array<File>` once files have been selected.

**`file-list.component.ts`**

```ts
import {
  Component,
  Input
} from '@angular/core';

@Component({
  selector: 'file-list',
  templateUrl: 'file-list.component.html'
})
export class FileListComponent {
  @Input() files: File[];
  @Input() layout = "row | wrap";
  @Input() align = "start start"
  @Input() elevated = true;
}
```  

In addition to the `File[]` input property, the `layout`, `align`, and `elevated` input properties can be provided to determine how the file list will be rendered.

**`file-list.component.html`**

```html
<p class="mat-title">Pending Uploads</p>
<section class="container"
         [fxLayout]="layout"
         [fxLayoutAlign]="align">
  <section *ngFor="let f of files"
           class="background card container"
           [class.static-elevation]="elevated">
    <p class="mat-subheading-2">{{f.name}}</p>
    <p>{{f.lastModified | date:'dd MMM yyyy HH:mm'}}</p>
    <p>{{f.type}}</p>
  </section>
</section>
```

Because we have access to the `Upload.fileType`, the card that is used to render an `Upload` can be dynamically configured to render the content of the `Upload` for certain file types. This way, we can render all `Upload` items using a single card component, but have them render according to their content type.

**`upload-card.component.ts`**

```ts
import {
  Component,
  Input,
  Output,
  EventEmitter,
  OnInit
} from '@angular/core';

import { Upload } from '../../models';

@Component({
  selector: 'upload-card',
  templateUrl: 'upload-card.component.html'
})
export class UploadCardComponent implements OnInit {
  expandable: boolean;
  filetype: string;
  @Input() expanded = false;
  @Input() clickable = true;
  @Input() upload: Upload;
  @Input() size = 600;
  @Output() select = new EventEmitter<Upload>();
  @Output() delete = new EventEmitter<Upload>();

  toggleExpanded = () => this.expanded = !this.expanded;

  ngOnInit() {
    this.filetype = this.upload.fileType.split('/')[0];

    switch (this.filetype) {
      case 'image':
      case 'audio':
      case 'video':
        this.expandable = true;
        break;
      default:
        this.expandable = false;
        this.expanded = false;
    }
  }
}
```  

Here is a description of the properties for this component:

Property | Type | Scope | Description
---------|------|-------|------------
`expandable` | `boolean` | local | Determines whether or not the card's content can be expanded for display
`filetype` | `string` | local | Represents the sub-type of the MIME type specified by `upload.fileType`
`expanded` | `boolean` | input | Whether or not the card's default state is expanded
`clickable` | `boolean` | input | Whether or not the title of the card should have the `.clickable` class applied
`upload` | `Upload` | input | The `Upload` instance the card represents
`size` | `number` | input | The width the card should be rendered at
`select` | `EventEmitter<Upload>` | output | Provides the `upload` property to indicate that it has been selected
`delete` | `EventEmitter<Upload>` | output | Provides the `upload` property to indicate that it has been deleted

The `toggleExpanded()` function simply toggles the state of the `expanded` property.

In the **OnInit** lifecycle hook, the `filetype` property is set based on the value of `upload.fileType`. Then, based on the content type represented by the value of `filetype`, the `expandable` property is set.

**`upload-card.component.html`**

```html
<section class="background card elevated"
         fxLayout="column"
         fxLayoutAlign="start stretch"
         [style.width.px]="size">
  <section class="container"
           fxLayout="row"
           fxLayoutAlign="start center">
    <p class="mat-title"
       [class.clickable]="clickable"
       fxFlex
       (click)="clickable && select.emit(upload)">{{upload.file}}</p>
    <button mat-icon-button
            color="warn"
            matTooltip="Delete"
            (click)="delete.emit(upload)">
      <mat-icon>delete</mat-icon>
    </button>
    <a mat-icon-button
       target="_blank"
       matTooltip="Download"
       [href]="upload.url">
      <mat-icon>save_alt</mat-icon>
    </a>
    <button *ngIf="expandable"
            mat-icon-button
            [matTooltip]="expanded ? 'Collapse' : 'Expand'"
            (click)="toggleExpanded()">
      <mat-icon *ngIf="expanded">keyboard_arrow_down</mat-icon>
      <mat-icon *ngIf="!(expanded)">keyboard_arrow_right</mat-icon>
    </button>
  </section>
  <section *ngIf="expanded">
    <ng-container [ngSwitch]="upload.fileType.split('/')[0]">
      <img *ngSwitchCase="'image'"
           [src]="upload.url"
           [alt]="upload.file"
           [width]="size">
      <section *ngSwitchCase="'audio'"
               class="container"
               fxLayout="column"
               fxLayoutAlign="center center">
        <audio controls
               [src]="upload.url"
               [style.width.px]="size - 20"></audio>
      </section>
      <video *ngSwitchCase="'video'"
             controls
             [width]="size">
        <source [src]="upload.url"
                [type]="upload.fileType">
        Sorry, this format isn't supported in your browser
      </video>
    </ng-container>
  </section>
  <mat-divider *ngIf="!(expanded)"></mat-divider>
  <section fxLayout="column"
           fxLayoutAlign="start center"
           class="container"
           [style.margin.px]="8">
    <mat-chip-list selectable="false">
      <mat-chip [matTooltip]="upload.fileType">{{upload.fileType | truncate:'15'}}</mat-chip>
      <mat-chip>{{upload.size | bytes}}</mat-chip>
      <mat-chip>{{upload.uploadDate | date:'dd MMM yyyy HH:mm'}}</mat-chip>
    </mat-chip-list>
  </section>
</section>
```  

The template for the card is represented by four sections:

* The base card
* The title bar
* The content region
* The upload metadata

**Base Card**  

This section of the template is represented by the root `<section>` element. The width of the card is set based on the `size` property.

**Title Bar**

This section of the template is represented by the first `<section>` element inside of the root `<section>`. It uses a row layout to render its content. The first item rendered is the title, represented by `upload.file`. If the `clickable` property is true, clicking the title will call `select.emit(upload)`, and apply the `.clickable` class to the title. The second item rendered is a delete button. If clicked, `delete.emit(upload)` is called. The third item rendered is a link button that resolves to `upload.url` in a new tab. If `expandable` is true, a toggle button is rendered which calls `toggleExpanded()` when clicked.

**Content Region**

This section of the template is represented by the second `<section>` element inside of the root `<section>`. If the `filetype` property is any of the following, the card is `expandable` and renders the content specified:

* **image** - An `<img>` element is rendered with the `src` pointed to `upload.url`. The width of the image is determined by the `size` property, and the `alt` text for the image is specified as `upload.file`.
* **audio** - An `<audio>` element is rendered with controls, and the `src` points to `upload.url`. The width of the audio controls are determined by the `size` property minus **20px**.
* **video** - A `<video>` element is rendered with controls, and the width is determined by the `size` property. Inside, a `<source>` element is defined specifying `src` as `upload.url`, and the `type` as `upload.fileType`. If the browser doesn't support this format, a message will be displayed in place of the video.

[NgSwitch](https://angular.io/guide/structural-directives#inside-ngswitch-directives) is used to determine what to render inside of the content region.

If the `expanded` property is false, a `MatDivider` is rendered in place of the content region.

**Upload Metadata**  

This section of the template is represented by the third `<section>` element inside of the root `<section>`. It uses a `MatChipList` to render the additional properties of the `Upload` the card represents. Not that the `upload.size` chip uses the `bytes` pipe to appropriately display the file size.

### [Dialogs](#uploads)

Because uploads can be soft-deleted, it's important to have a way to easily access deleted uploads to either restore them, or permanently remove them. This is accomplished with an `UploadBinDialog` component.

> If the term dialog is unfamiliar to you, refer to the [Dialogs](./a3-dialogs.md) article.

### [Route](#uploads)

## [Related Data](#uploads)

[Back to Top](#uploads)