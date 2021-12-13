
# Avro [![build](https://github.com/c-cube/ocaml-avro/actions/workflows/main.yml/badge.svg)](https://github.com/c-cube/ocaml-avro/actions/workflows/main.yml)

- `avro` is the runtime library, should be reasonably small (depends only on `camlzip`)
- `avro-compiler` can read a json schema and generate code from it

The online documentation can be found [here](https://c-cube.github.io/ocaml-avro/)

## Supported features

- [x] code generation from json schema (using `avro-compiler`)
  * **NOTE**: default values are not supported yet.
- [x] object container file format
- [x] binary encoding
- [ ] schema evolution
  (in particular the schema is emitted as-is, nothing is stripped)
- [ ] single object file format
- [ ] sorting
- [ ] json encoding
- compression codecs:
  * [x] null
  * [x] deflate
  * [ ] snappy (optional codec)

## Schemas

Schemas are json files
(described [here](https://avro.apache.org/docs/1.7.6/spec.html#schemas)).
Some examples can be found in `tests/`.

For example, the schema:

```json
{
  "name": "employee",
  "type": "record",
  "fields": [
    {"name": "firstname", "type": "string"},
    {"name": "lastname", "type": "string"},
    {"name": "age", "type": "int"},
    {"name": "friendliness", "type": "float"},
    {"name": "job", "type": [
      {"name": "programmer", "type": "record",
        "fields": [{"name": "language", "type": {"type":"enum", "name":"language","symbols": ["cpp", "ocaml", "java", "python"]}}]},
      {"name": "RH", "type": "record",
        "fields": [{"name": "hair_spikiness", "type": {"type":"fixed", "size": 4, "name":"spikiness"}}]}
    ] }
  ]
}
```

will be compiled to the types:

```ocaml
type nonrec language =   | Cpp | Ocaml | Java | Python

type nonrec programmer = { language: language; }

type nonrec rH = { hair_spikiness: string; }
type nonrec union_0 =  
  | C_programmer of programmer 
  | C_RH of rH 
                       
type nonrec t = {
  firstname: string; 
  lastname: string; 
  age: int; 
  friendliness: float; 
  job: union_0; 
}
```

## Compression

New codecs can be registered in `Avro.Obj_container_format.Codec`.
Currently it's quite naive as it goes from a string to a string, instead of
being fully streaming. This might change before 1.0.


```ocaml
module Codec : sig
  type t

  (* … *)

  val register :
    name:string ->
    compress:(string -> string) ->
    decompress:(string -> string) ->
    unit -> t
  (** Register decompression codecs. Defaults are "null" and "deflate". *)
end
```

## Examples

See some examples from the `tests/` directory:

- `tests/records.json` and `tests/records_test.ml` that use it.
  you can see what the schema compiler will produce:

  ```sh
  $ ./compiler.sh ./tests/records.json
  (* generated by avro-compiler *)

  open Avro

  module Str_map = Map.Make(String)

  let schema = "{\"type\":\"record\",\"name\":\"test\",\"fields\":[{\"type\":\"long\",\"name\":\"a\"},{
  \"type\":\"string\",\"name\":\"b\"}]}"

  type nonrec t = { a: int64;  b: string; }

  let read (input:Input.t) : t =
    (let a = Input.read_int64 input in
     let b = Input.read_string input in
     { a;b })

  let write (out:Output.t) (self:t) : unit =
    ((let self = self.a in Output.write_int64 out self);
      (let self = self.b in Output.write_string out self);
      )

  ```

- similarly for `tests/employee.json` and `tests/employee_test.ml` that use it

## License 

MIT license.
