# Laboratorio de BDD con Selenium, ChromeDriver y .NET Core en GitHub Codespaces

## 1. Crear un nuevo repositorio en GitHub
- Ve a GitHub y crea un nuevo repositorio.
- Dale un nombre explícito, por ejemplo `bdd-netcore-codespaces`.
- Agrega una descripción a tu repositorio (opcional).

## 2. Configurar GitHub Codespaces
Para poder programar en .NET desde el navegador o VS Code en Codespaces y usar Chrome en las pruebas, configuraremos un DevContainer.

- En tu repositorio, haz clic en **Add file > Create new file**.
- En el nombre del archivo ingresa `.devcontainer/devcontainer.json`.
- Pega la siguiente configuración para instalar .NET, NodeJS (opcional pero útil) y las herramientas de Google Chrome:

```json
{
  "name": "BDD Lab with Selenium and .NET Core",
  "image": "mcr.microsoft.com/devcontainers/dotnet:8.0",
  "features": {
    "ghcr.io/devcontainers/features/node:1": {
      "version": "lts"
    },
    "ghcr.io/kreemer/features/chrometesting:1": {}
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "ms-dotnettools.csharp",
        "ms-dotnettools.vscode-dotnet-runtime",
        "techtalk.specflow-vscode"
      ]
    }
  },
  "forwardPorts": [5000, 5001],
  "postCreateCommand": "sudo apt update"
}
```

- Escribe un mensaje de commit y haz clic en **Commit changes...**.

## 3. Crear el Codespace
- Vuelve a la página principal de tu repositorio haciendo clic en `<> Code` (esquina superior izquierda).
- Haz clic en el botón verde `<> Code`, cambia a la pestaña **Codespaces** y haz clic en **Create codespace on main** (o el botón **+**).
- Espera a que termine la inicialización del contenedor y se abra el editor.

## 4. Creación del Proyecto de Pruebas con .NET Core
Para desarrollar el laboratorio, primero debemos crear un proyecto de pruebas en .NET. Para este BDD utilizaremos NUnit.
Abre una terminal en tu entorno (Terminal > New Terminal) y ejecuta los siguientes comandos:

```bash
dotnet new nunit -n BddDotNetLab
cd BddDotNetLab
```

## 5. Agregar dependencias de SpecFlow y Selenium
Para que nuestras pruebas funcionen y utilicen Chrome, necesitamos agregar varios paquetes `NuGet` a nuestro proyecto. Ejecuta en la terminal:

```bash
dotnet add package SpecFlow.NUnit
dotnet add package Selenium.WebDriver
dotnet add package Selenium.WebDriver.ChromeDriver
```

## 6. Crear la estructura BDD
Dentro del proyecto, limpiaremos lo que no vamos a usar y prepararemos la estructura para BDD.

- Borra el archivo de ejemplo `UnitTest1.cs` que se generó por defecto:
```bash
rm UnitTest1.cs
```

- Dentro de la carpeta del proyecto `BddDotNetLab`, crea las siguientes carpetas:
  - `Features`
  - `Steps`

Tu estructura de proyecto debería verse parecida a esta:
```text
BddDotNetLab/
 ├─ Features/
 ├─ Steps/
 ├─ BddDotNetLab.csproj
 └─ Usings.cs
```

## 7. Crear el escenario BDD (Feature)
Vamos a definir el comportamiento esperado de nuestro sistema simulando una búsqueda en Google.

- Dentro de la carpeta `Features`, crea un nuevo archivo llamado `GoogleSearch.feature`.
- Agrega el siguiente texto en formato Gherkin:

```feature
Feature: Google Search

Scenario: Search for a term
  Given I am on the Google search page
  When I search for "GitHub"
  Then I should see "GitHub" in the results
```

## 8. Automatizar los pasos de prueba (Step Definitions)
Ahora escribiremos el código en C# que interpreta los pasos definidos en el archivo de texto y ejecuta órdenes en un navegador de verdad (usando Selenium).

- Dentro de la carpeta `Steps`, crea un archivo llamado `SearchSteps.cs`.
- Pega el siguiente código:

```csharp
using NUnit.Framework;
using OpenQA.Selenium;
using OpenQA.Selenium.Chrome;
using TechTalk.SpecFlow;

namespace BddDotNetLab.Steps
{
    [Binding]
    public class SearchSteps
    {
        private IWebDriver _driver = null!;

        [BeforeScenario]
        public void SetUp()
        {
            var options = new ChromeOptions();
            // Ejecutar Chrome/Chromium en modo silencioso (headless) esencial para Codespaces
            options.AddArgument("--headless");
            options.AddArgument("--disable-gpu");
            options.AddArgument("--no-sandbox"); // Necesario para correr dentro de Docker/Codespaces
            options.AddArgument("--disable-dev-shm-usage");
            options.AddArgument("--remote-allow-origins=*");

            // Indicamos la ruta del ChromeDriver instalado por la feature "chrometesting" del DevContainer
            _driver = new ChromeDriver("/usr/local/bin", options);
            
            // Espera implícita para que la página renderice los elementos
            _driver.Manage().Timeouts().ImplicitWait = System.TimeSpan.FromSeconds(10);
        }

        [Given(@"I am on the Google search page")]
        public void GivenIAmOnTheGoogleSearchPage()
        {
            _driver.Navigate().GoToUrl("https://www.google.com");
        }

        [When(@"I search for ""(.*)""")]
        public void WhenISearchFor(string term)
        {
            var searchBox = _driver.FindElement(By.Name("q"));
            searchBox.SendKeys(term);
            searchBox.Submit();
        }

        [Then(@"I should see ""(.*)"" in the results")]
        public void ThenIShouldSeeInTheResults(string term)
        {
            // Esperar a que Google termine de cargar los resultados dinámicamente
            System.Threading.Thread.Sleep(2000);

            string pageSource = _driver.PageSource;
            Assert.IsTrue(pageSource.Contains(term), $"No se encontró el término '{term}' en la página.");
        }

        [AfterScenario]
        public void TearDown()
        {
            _driver.Quit();
            _driver.Dispose();
        }
    }
}
```

## 9. Ejecutar las pruebas
- Asegúrate de haber guardado todos los archivos.
- En la terminal de tu Codespace, asegúrate de estar dentro del directorio `BddDotNetLab` y ejecuta:

```bash
dotnet test
```

Verás una salida en la terminal indicando que la prueba se construyó y pasó exitosamente (indicando la cantidad de pruebas ejecutadas: **1 Pasada**).
