---
name: Carga de archivos
weight: 11
aliases:
  - /docs/file-uploads
  - /es/docs/file-uploads
---

# Carga de archivos

{{< since "0.10.3" >}}

Buffalo permite un manejo más fácil de archivos cargados desde un formulario. Almacenar estos archivos, tanto en el disco o S3, depende de ti, el desarrollador final: Buffalo solo te brinda el fácil acceso al archivo desde la solicitud.

## Configuración del formulario

El helper de formulario `f.FileTag` se puede usar para agregar rápidamente un archivo al formulario. Al usar esto, el `enctype` del formulario cambia **automaticamente** a `multipart/form-data`.

```html
<%= form_for(widget, {action: widgetsPath(), method: "POST"}) { %>
  <%= f.InputTag("Name") %>
  <%= f.FileTag("MyFile", {name: "someFile"}) %>
  <button class="btn btn-success" role="submit">Save</button>
  <a href="<%= widgetsPath() %>" class="btn btn-warning" data-confirm="Are you sure?">Cancel</a>
<% } %>
```

## Accediendo al archivo del formulario

[`buffalo.Context`](https://godoc.org/github.com/gobuffalo/buffalo#Context) tiene un método `c.File` que toma una cadena como parámetro, el `name` del input de archivo del formulario, y devolverá un [`binding.File`](https://godoc.org/github.com/gobuffalo/buffalo/binding#File) que se puede usar para recuperar fácilmente un archivo desde este.

```go
func SomeHandler(c buffalo.Context) error {
  // ...
  f, err := c.File("someFile")
  if err != nil {
    return errors.WithStack(err)
  }
  // ...
}
```

## Mapeando a una estructura

El [`c.Bind`](https://godoc.org/github.com/gobuffalo/buffalo#Context) permite vincular elementos del formulario a una estructura, pero también puede adjuntar archivos cargados a la estructura. Para esto, el tipo del campo de la estructura **DEBE** ser de tipo `binding.File`

En el siguiente ejemplo podrás ver un modelo, el cual está configurado para tener un campo `MyFile` de tipo `binding.File`. Hay un callback `AfterCreate` en este modelo de ejemplo que guarda el archivo en el disco después que el modelo se haya guardado correctamente en la base de datos.

```go
// models/widget.go
type Widget struct {
  ID        uuid.UUID    `json:"id" db:"id"`
  CreatedAt time.Time    `json:"created_at" db:"created_at"`
  UpdatedAt time.Time    `json:"updated_at" db:"updated_at"`
  Name      string       `json:"name" db:"name"`
  MyFile    binding.File `db:"-" form:"someFile"`
}

func (w *Widget) AfterCreate(tx *pop.Connection) error {
  if !w.MyFile.Valid() {
    return nil
  }
  dir := filepath.Join(".", "uploads")
  if err := os.MkdirAll(dir, 0755); err != nil {
    return errors.WithStack(err)
  }
  f, err := os.Create(filepath.Join(dir, w.MyFile.Filename))
  if err != nil {
    return errors.WithStack(err)
  }
  defer f.Close()
  _, err = io.Copy(f, w.MyFile)
  return err
}
```

{{<note>}}
El campo `MyFile` no se guarda en la base de datos debido al tag de estructura `db:"-"`.
{{</note>}}


## Probando la carga de archivos

La librería de prueba de HTTP [`github.com/gobuffalo/httptest`](https://github.com/gobuffalo/httptest) (incluida en el paquete [`github.com/gobuffalo/suite`](https://github.com/gobuffalo/suite) que usa Buffalo para pruebas) ha sido actualizado para incluir dos nuevas funciones: [`MultiPartPost`](https://godoc.org/github.com/gobuffalo/httptest#Request.MultiPartPost) y [`MultiPartPut`](https://godoc.org/github.com/gobuffalo/httptest#Request.MultiPartPut).


Estos métodos funcionan como los métodos `Post` y `Put`, pero en su lugar, envía formularios `multipart`, y pueden aceptar archivos para cargar.

Al igual que `Post` y `Put`; `MultiPartPost` and `MultiPartPut`, toman una estructura o mapa como el primer argumento: Esto es el equivalente al formulario de HTML que se enviará. Los metodos toman un segundo armumento variable [`httptest.File`](https://godoc.org/github.com/gobuffalo/httptest#File).

Un `httptest.File` requiere el nombre del parámetro del formulario, `ParanName`; el nombre del archivo, `FileName`; y un `io.Reader`, presumiblemente el archivo que deseas cargar.


{{< codetabs >}}
{{< tab "actions/widgets_test.go" >}}
```go
// actions/widgets_test.go

func (as *ActionSuite) Test_WidgetsResource_Create() {
  // clear out the uploads directory
  os.RemoveAll("./uploads")

  // setup a new Widget
  w := &models.Widget{Name: "Foo"}

  // find the file we want to upload
  r, err := os.Open("./logo.svg")
  as.NoError(err)
  // setup a new httptest.File to hold the file information
  f := httptest.File{
    // ParamName is the name of the form parameter
    ParamName: "someFile",
    // FileName is the name of the file being uploaded
    FileName: r.Name(),
    // Reader is the file that is to be uploaded, any io.Reader works
    Reader: r,
  }

  // Post the Widget and the File(s) to /widgets
  res, err := as.HTML("/widgets").MultiPartPost(w, f)
  as.NoError(err)
  as.Equal(302, res.Code)

  // assert the file exists on disk
  _, err = os.Stat("./uploads/logo.svg")
  as.NoError(err)

  // assert the Widget was saved to the DB correctly
  as.NoError(as.DB.First(w))
  as.Equal("Foo", w.Name)
  as.NotZero(w.ID)
}
```
{{< /tab >}}

{{< tab "actions/widgets.go" >}}
```go
// actions/widgets.go

// Create adds a Widget to the DB. This function is mapped to the
// path POST /widgets
func (v WidgetsResource) Create(c buffalo.Context) error {
  // Allocate an empty Widget
  widget := &models.Widget{}

  // Bind widget to the html form elements
  if err := c.Bind(widget); err != nil {
    return errors.WithStack(err)
  }

  // Get the DB connection from the context
  tx, ok := c.Value("tx").(*pop.Connection)
  if !ok {
    return errors.WithStack(errors.New("no transaction found"))
  }

  // Validate the data from the html form
  verrs, err := tx.ValidateAndCreate(widget)
  if err != nil {
    return errors.WithStack(err)
  }

  if verrs.HasAny() {
    // Make widget available inside the html template
    c.Set("widget", widget)

    // Make the errors available inside the html template
    c.Set("errors", verrs)

    // Render again the new.html template that the user can
    // correct the input.
    return c.Render(422, r.HTML("widgets/new.html"))
  }

  // If there are no errors set a success message
  c.Flash().Add("success", "Widget was created successfully")

  // and redirect to the widgets index page
  return c.Redirect(302, "/widgets/%s", widget.ID)
}
```
{{< /tab >}}
{{< tab "models/widgets.go" >}}
```go
// models/widgets.go

package models

import (
  "encoding/json"
  "io"
  "os"
  "path/filepath"
  "time"

  "github.com/gobuffalo/buffalo/binding"
  "github.com/gobuffalo/pop"
  "github.com/markbates/validate"
  "github.com/markbates/validate/validators"
  "github.com/pkg/errors"
  "github.com/satori/go.uuid"
)

type Widget struct {
  ID        uuid.UUID    `json:"id" db:"id"`
  CreatedAt time.Time    `json:"created_at" db:"created_at"`
  UpdatedAt time.Time    `json:"updated_at" db:"updated_at"`
  Name      string       `json:"name" db:"name"`
  MyFile    binding.File `db:"-" form:"someFile"`
}

// String is not required by pop and may be deleted
func (w Widget) String() string {
  jw, _ := json.Marshal(w)
  return string(jw)
}

// Widgets is not required by pop and may be deleted
type Widgets []Widget

// String is not required by pop and may be deleted
func (w Widgets) String() string {
  jw, _ := json.Marshal(w)
  return string(jw)
}

func (w *Widget) AfterCreate(tx *pop.Connection) error {
  if !w.MyFile.Valid() {
    return nil
  }
  dir := filepath.Join(".", "uploads")
  if err := os.MkdirAll(dir, 0755); err != nil {
    return errors.WithStack(err)
  }
  f, err := os.Create(filepath.Join(dir, w.MyFile.Filename))
  if err != nil {
    return errors.WithStack(err)
  }
  defer f.Close()
  _, err = io.Copy(f, w.MyFile)
  return err
}

// Validate gets run every time you call a "pop.Validate*" (pop.ValidateAndSave, pop.ValidateAndCreate, pop.ValidateAndUpdate) method.
// This method is not required and may be deleted.
func (w *Widget) Validate(tx *pop.Connection) (*validate.Errors, error) {
  return validate.Validate(
    &validators.StringIsPresent{Field: w.Name, Name: "Name"},
  ), nil
}

// ValidateCreate gets run every time you call "pop.ValidateAndCreate" method.
// This method is not required and may be deleted.
func (w *Widget) ValidateCreate(tx *pop.Connection) (*validate.Errors, error) {
  return validate.NewErrors(), nil
}

// ValidateUpdate gets run every time you call "pop.ValidateAndUpdate" method.
// This method is not required and may be deleted.
func (w *Widget) ValidateUpdate(tx *pop.Connection) (*validate.Errors, error) {
  return validate.NewErrors(), nil
}
```

{{< /tab >}}
{{< /codetabs >}}
