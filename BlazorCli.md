# Build Blazor page/component and publish it to npm/nuget

> TL; DR: this is more like feature proposal for future Blazor Cli

Taking inspiration from Vue that can [build SFC (Single File Component) `*.vue`](https://github.com/vuejs/vue-cli/blob/dev/docs/cli.md#vue-build) into redistributable [package](https://github.com/vuejs/vue-cli/blob/dev/docs/build-targets.md#library), it would be nice if Blazor can do the same thing. Because Blazor [can be compiled into](http://blog.stevensanderson.com/2018/02/06/blazor-intro/) `*.dll` (AOT compiled mode) and `.wasm` (interpreted mode), it also make sense to have ability to build and publish single Blazor Component (`.cshtml`) into NuGet and/or NPM for shareabilitiy.

<details open>
<summary>Given this Blazor Component (if in the future Blazor support F#)</summary>

###### with usage

```html
<Propeller diameter="120cm" power="24W" propconstant="3.2" scale="0.5px/cm" />
```

###### then the implementation of `Propeller.fshtml` will be

```fs
<!-- https://codepen.io/drsensor/pen/ELEzWb -->
<svg width="@Diameter" height="@Diameter">
  <g transform="translate(0 @(Diameter/4))">
    <rect width="@Diameter" height="@(Diameter/2)" rx="@(Diameter/2)" ry="@(Diameter/4)" >
          <animateTransform attributeName="transform"
                          type="rotate"
                          from="0 @(Diameter/2) @(Diameter/4)"
                          to="360 @(Diameter/2) @(Diameter/4)"
                          dur="@Period"
                          repeatCount="indefinite"/>
    </rect>
    <rect width="@Diameter" height="@(Diameter/2)" rx="@(Diameter/2)" ry="@(Diameter/4)" >
      <animateTransform attributeName="transform"
                        type="rotate"
                        from="90 @(Diameter/2) @(Diameter/4)"
                        to="450 @(Diameter/2) @(Diameter/4)"
                        dur="@Period"
                        repeatCount="indefinite"/>
    </rect>
  </g>
  <text x="25%" y="10%">@Rpm RPM</text>
  <text x="75%" y="90%">@Thrust N</text>
</svg>

@functions {
  /// https://quadcopterproject.wordpress.com/static-thrust-calculation/
  [<Measure>] type rad (* revolution or radians *)
  [<Measure>] type s (* second *)
  [<Measure>] type m (* meter *)
  [<Measure>] type px (* pixels *)
  [<Measure>] type kg (* kilogram *)
  [<Measure>] type N = (kg*m)/(s^2) (* Newtons *)
  [<Measure>] type W = (N*m)*(rad/s) (* Watts *) (* P = τ * ω *)
  let π = 3.14159265359
  let HertzOf (rpm : <rad/s>) : float<rad/s> = rpm/60

  [<Measure>] let cm = 1/100<m>

  let public scale : float<px/m> = 1.0<px/m>
  val public power : float<W>
  val public diameter : float<m>
  val public propconstant : float<N*m>

  let public propRpm powerFactor : float<rad/s> = (power/propconstant)^(1/powerFactor)

  member Rpm with get () = propRpm(3.2)
  member Thrust : float<N>
    with get () = ((π/2) * (diameter^2) * 1.225<kg/m^3> * (power^2) )^(1/3)

  member Period : float<script> with get () = 1/(HertzOf(Rpm))
  member Diameter : int<px> with get () = diameter*scale
}
```
</details>

## Build and Publish to NPM

Taking consideration that WebComponent became widely adopted, there is 2 build target that essential for sharing Blazor Component into another JS/TS based only project.

### Build as WebAssembly

> not sure about this

#### CLI

```bash
dotnet blazor build --target wasm Propeller.fshtml
cd dist
npm version minor
npm publish
```

<details>
<summary>For building multiple Blazor Component into single wasm (not usre if this good)</summary>

given the `Propeller.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
    <EnableDefaultItems>false</EnableDefaultItems>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.FSharp" Version="*" />
    <Compile Include="PropellerAPC.fshtml" />
    <Compile Include="PropellerWD.cshtml" />
    <Compile Include="PropellerCF.cshtml" />
  </ItemGroup>
</Project>
```

then build the `.csproj` file

```bash
dotnet blazor build --target wasm Propeller.csproj
```

> Still don't know if it's best to output single `Propeller.wasm` or multiple `Propeller*.wasm` ?

</details>

#### Usage

* using webpack and wasm-loader

> waiting for docs how to use _webpack 4_ `webassembly/experimental`

```ts
import propeller from 'propeller'

propeller().then(instance => {
  // https://quadcopterproject.wordpress.com/static-thrust-calculation/
  const calcrpm = instance.exports.propRpm
  const rpm = calcrpm(3.2)
  console.log(`${rpm} RPM`) // 24 Watt
})
```

* using [fetch](https://developer.mozilla.org/en-US/docs/WebAssembly/Loading_and_running) (usefull for dynamic loading at runtime via CDN)

```html
<button onclick="instantiatePropeller()">load component</button>
<p id="rpm" />
<script>
function instantiatePropeller() {
  WebAssembly.instantiateStreaming(
    fetch('https://cdn.jsdelivr.net/npm/propeller/dist/propeller.wasm'), importObject)
    .then(obj => {
      document.getElementById('rpm').textContent = obj.instance.exports.calcrpm(3.2);
    }
  );
}
</script>
```

### Build as WebComponent

> If Blazor heading toward to support WebComponent

The produced WebComponent are just binding to the generated WebAssembly.

#### CLI

```bash
dotnet blazor build --target wc Propeller.fshtml
cd dist
npm version minor
npm publish
```

#### Usage

- via CDN

```html
<head>
  <script src="https://cdn.jsdelivr.net/npm/propeller" />
</head>
<body>
  <propeller diameter="120cm" power="24W" propconstant="3.2" scale="0.5px/cm" />
</body>
```

<details>
<summary>For building multiple Blazor Component into single `propeller.min.js`</summary>

> the `Propeller.csproj` same as wasm

```bash
dotnet blazor build --target wc Propeller.csproj
```

using multiple component

```html
<head>
  <script src="https://cdn.jsdelivr.net/npm/propeller" />
</head>
<body>
  <propeller-apc diameter="120cm" power="24W" scale="0.5px/cm" />
  <propeller-w diameter="60cm" power="24W" scale="0.5px/cm" />
  <propeller-cf diameter="80cm" power="24W" scale="0.5px/cm" />
</body>
```

</details>

## Build and Publish to NuGet

While dotnet cli can build standalone blazor component, it would be nice to have dedicated cli for building standalone blazor component without tinkering with `.csproj` file.

### CLI

```bash
dotnet blazor build Propeller.fshtml
cd nupkgs
dotnet nuget push Propeller.nupkg
```

### Usage

adding to `.csproj`

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.Blazor.Browser" Version="*" />
    <PackageReference Include="Microsoft.AspNetCore.Blazor.Build" Version="*" />
    <PackageReference Include="Propeller" Version="*" />
  </ItemGroup>
</Project>
```

In `PropellerAPC.cshtml`

```cshtml
@addTagHelper *, Propeller
@using Propeller

<Propeller power="@power" scale="@scale" diameter="@diameter" propconstant="4.6" />

@functions {
  public double scale { get; set; } = 1.0;
  public double power { get; set; }
  public double diameter { get; set; }
}
```
