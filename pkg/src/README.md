# src

src is a specialized but generic code generator to support a few use cases elegantly. The API is intentionally not
created to reflect any facette or possibility of a specific language. It is more comparable to a transpiler, because it
can emit high quality Go (Golang) and Java source.

## possible alternatives

* Daves [jennifer](https://github.com/dave/jennifer)
* go [text template](https://golang.org/pkg/text/template/)
* a big [list](https://github.com/golang/go/wiki/GoGenerateTools) of many *go generate* tools
* https://elixir-lang.org/

## What is so special?
* There are a few things, which usually do not belong to an AST
  * parent-child relations
  * extra terminator symbols
* AST Specification cannot only express Go code, but can also represent a mix of other languages
* Macros which can produce language specific and context dependent AST children
  to express higher level operations in a cross platform way (e.g. try logic).
  Macros can evaluate their context and emit type safe code.
* defining a standard library whose types are translated by the concrete language renderer
* multiple modules in different languages can be represented within the same project tree.

## example

```go
package golang

import (
	"fmt"
	. "github.com/worldiety/macro/pkg/src/ast"
	"github.com/worldiety/macro/pkg/src/stdlib"
	fmt2 "github.com/worldiety/macro/pkg/src/stdlib/fmt"
	"github.com/worldiety/macro/pkg/src/stdlib/lang"
	"testing"
)

func TestRenderer_Render(t *testing.T) {
	prj := newProject()
	renderer := NewRenderer(Options{})
	artifact, err := renderer.Render(prj)
	if err != nil {
		fmt.Println(artifact)
		t.Fatal(err)
	}

	fmt.Println(artifact)
}

func newProject() *Prj {
	preamble := "Code generated by golangee/architecture. DO NOT EDIT."

	return NewPrj("MyEpicProject").
		AddModules(
			NewMod("github.com/myproject/mymodule").
				SetLang(LangGo).
				AddPackages(
					NewPkg("github.com/myproject/mymodule/cmd/myapp").
						SetName("main").
						SetPreamble(preamble).
						SetComment("...is the actual package doc.").
						AddFiles(
							NewFile("main.go").
								SetPreamble(preamble).
								SetComment("...is a funny package.").
								AddTypes(
									NewStruct("HelloWorld").
										SetComment("... shows a struct.").
										AddFields(
											NewField("Hello", NewSimpleTypeDecl(stdlib.String)).
												SetComment("...holds a hello string."),
											NewField("World", NewSimpleTypeDecl(stdlib.String)).
												SetComment("...holds a world string.").
												AddAnnotations(
													NewAnnotation("json").SetDefault("world"),
													NewAnnotation("db").SetDefault("hello_world"),
												),
										).
										AddMethods(
											NewFunc("SayHello").
												SetComment("...shouts it into the world.").
												SetBody(
													NewBlock(
														NewBlock().SetComment("this is a redundant block"),
													),
												),
											NewFunc("Hello2").
												SetComment("...is a more complex method.").
												AddParams(
													NewParam("hey", NewSimpleTypeDecl(stdlib.Int)).SetComment("...declares a number."),
													NewParam("ho", NewSimpleTypeDecl(stdlib.Float64)).SetComment("...declares a float."),
												).
												AddResults(
													NewParam("", NewSliceTypeDecl(NewSimpleTypeDecl(stdlib.String))).SetComment("...a list of strings."),
													NewParam("", NewSimpleTypeDecl(stdlib.String)).SetComment("...declares a number."),
													NewParam("", NewSimpleTypeDecl(stdlib.Error)).SetComment("...is returned if everything fails."),
												).
												SetRecName("h").
												SetBody(NewBlock(
													fmt2.Println(NewIdent("hey"), NewIdent("ho"), NewStrLit("hello world")), lang.Term(),
													lang.TryDefine(NewIdent("rows"), lang.CallStatic("sql.query"), "cannot query"),
												)),
										),
								).
								AddFuncs(
									NewFunc("globalFunc").
										SetComment("...is a package private function.").
										SetVisibility(PackagePrivate).
										SetBody(NewBlock()),
								),
						),
				),

		)
}

```

Generated output may look like this:
```
 github.com[application/x-directory]:
    myproject[application/x-directory]:
      mymodule[application/x-directory-module]:
        cmd[application/x-directory]:
          myapp[application/x-directory]:
            doc.go[text/x-go-source]:
              // Code generated by golangee/architecture. DO NOT EDIT.
              
              // Package main is the actual package doc.
              package main
              
            main.go[text/x-go-source]:
              // Code generated by golangee/architecture. DO NOT EDIT.
              
              // Package main is a funny package.
              package main
              
              import fmt "fmt"
              import sql "sql"
              
              // HelloWorld shows a struct.
              type HelloWorld struct {
              	// Hello holds a hello string.
              	Hello string
              
              	// World holds a world string.
              	World string `json:"world" db:"hello_world"`
              }
              
              // SayHello shouts it into the world.
              func (_ HelloWorld) SayHello() {
              	// this is a redundant block
              	{
              	}
              }
              
              // Hello2 is a more complex method.
              //
              // The parameter hey declares a number.
              // The parameter ho declares a float.
              // The result string declares a number.
              // The result error is returned if everything fails.
              func (h HelloWorld) Hello2(hey int, ho float64) ([]string, string, error) {
              	fmt.Println(hey, ho, "hello world")
              	rows, err := sql.query()
              	if err != nil {
              		return nil, "", fmt.Errorf("cannot query: %w", err)
              	}
              }
              
              // globalFunc is a package private function.
              func globalFunc() {
              }
```