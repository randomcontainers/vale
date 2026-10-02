# Extra license files

Two Go modules that vale links compile C sources whose licenses are not in the module archive. Each `<module>@<version>/` directory here has those files, laid out like the module: the license of a grammar is in the directory of its Go package. `scripts/go-licenses.sh` copies them into the image next to the module's own license files. The build stops when vale links another version of one of these modules, or a cgo package of it without a file in its directory; then update the files from the sources below and rename the directory. It also stops on a cgo package from a module that has no directory here, unless the module's own license covers its C sources and a `cgo-covered` file here lists its `<module>@<version>`.

| Files | Source |
|---|---|
| `github.com/smacker/go-tree-sitter@*/LICENSE.tree-sitter` | `LICENSE` of [tree-sitter](https://github.com/tree-sitter/tree-sitter) at the tag named by `sitterVersion` in go-tree-sitter's `_automation/treesitter_updater/main.go` (v0.22.5) |
| `github.com/smacker/go-tree-sitter@*/LICENSE.unicode` | `lib/src/unicode/LICENSE` of tree-sitter at the same tag, for the ICU headers in the runtime |
| `github.com/smacker/go-tree-sitter@*/<package>/` | the license files of the grammar repository at the revision listed for the package in go-tree-sitter's `_automation/grammars.json` |
| `github.com/jdkato/go-tree-sitter-julia@v0.1.0/LICENSE` | `LICENSE` of [go-tree-sitter-julia](https://github.com/jdkato/go-tree-sitter-julia), added after v0.1.0 in commit cd557502260aef9050c929232b9c694e0daba70a |
| `github.com/jdkato/go-tree-sitter-julia@v0.1.0/julia/LICENSE` | `LICENSE` of [tree-sitter-julia](https://github.com/tree-sitter/tree-sitter-julia), unchanged since 2020 |

The Lua grammar's repository has no license file; `lua/NOTE` says what it declares instead.

`LICENSE.en_US-web` is the `LICENSE` of [en_US-web](https://github.com/vale-cli/en_US-web), the American English dictionary that vale embeds from `internal/spell/data/`. vale's repository does not include it, so the build copies it from here.
