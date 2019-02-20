# Unit and Integration Testing on Angular 

 Il est important de comprendre le test sous Angular pour l'appréhender sous Ionic. Voici donc  un apperçu sous Angular.



doc: **fosdem**



## Setup



Angular CLI télécharge tout ce dont nous aurons besoin pour les tests avec le framework Jasmine.

```
ng test
```

Cette commande surveille le moindre changement (comme serve)

Le test lancera Karma test runner qui rendra la lecture des logs plus simple.

### **Configuration**



Le CLI s'occupe de la conf Karma et Jasmine. On peut néanmoins ajouter des potions ou les modifier dans Karma.conf.js et test.ts dans src/ .



### **Test File Name and location**



Dans app/src/

Le CLI génère un fichier app.component.spec.ts pour App.component et fait de même pour chaque component, page, service... présent dans notre /app .



## Set up Continuous integration

ex de services:

- Payants: 
  - Travis Ci (GLPI Fusion)
  - Circle CI (GLPI)
- Gratuits: 
  - Jenkins



Le meilleur moyen de s'assurer que notre code reste de créer des suites de test. Les serveurs d'intégration continue nous permettent de de paramétrer le dépôt du projet pour qu'un test se lance à chaque commit et pull request.

### **Configure CLI for CI testing in Chrome**

Bien que les commandes CLI 

```shell
ng test
```

et 

```shell
ng e2e
```

lancent en général les test CI dans notre environnement, il nous faut tout de même ajuster notre configuration pour lancer les tests dans le navigateur **Chrome**.

Il existe un fichier de configuration autant pour [Karma Javascript Test Runner](https://karma-runner.github.io/latest/config/configuration-file.html)  et [Protractor end-to-end testing tool](https://www.protractortest.org/#/api-overview)  qu'il faudra simplement ajuster pour démarrer Chrome.

Nous utiliserons [headless Chrome](https://developers.google.com/web/updates/2017/04/headless-chrome#cli) dans ces exemples.

* Dans le fichier de configuration Karma, karma.conf.js, on ajoute un launcher personnalisé appelé ChromeHeadlessCI en dessous de browsers: 

  ```javascript
  browsers: ['Chrome'],
  customLaunchers: {
    ChromeHeadlessCI: {
      base: 'ChromeHeadless',
      flags: ['--no-sandbox']
    }
  },
  ```

* Dans le dossier racine du projet, on crée un nouveau filchier appelé protractor-ci.conf.js. Ce nouveau fichier extends du protracor-ci.conf.js de base.

```js
const config = require('./protractor.conf').config;

config.capabilities = {
  browserName: 'chrome',
  chromeOptions: {
    args: ['--headless', '--no-sandbox']
  }
};

exports.config = config;
```

A présent, nous pouvons lancer les commandes suivantes en utlisant le --no-sandbox flag:

```shell
ng test -- --no-watch --no-progress --browsers=ChromeHeadlessCI
ng e2e -- --protractor-config=e2e/protractor-ci.conf.js
```

### Enable code coverage reports

Le CLI peut lancer des tests unitaires et créer un _Code Coverage Reports_ . Le Code Coverage Reports nous montre tout nos morceaux de code qui ne seront pas testés par notre test unitaire.



Pour générer un Coverage Code Report, il faut lancer la commande suivante depuis la base du projet:

```shell
ng test --no-watch --code-coverage
```

Si nous voulons créer un Code Coverage Report à chaque test lancé, on peut ajouter une option au fichier de conf angular.json: 

```javascript
"test": {
  "options": {
    "codeCoverage": true
  }
}
```

### Code Coverage Enforcement

Le _Code Coverage Percentages_ permet d'estimer combien de notre code est testé. Si nous décidons de mettre une quantité minimum de code à tester unitairement, on peut imposer ce minimum à Angular CLI.

Par exemple, imaginons vouloir que notre code soit testé à minimum 80%. Pour ce faire,  on ouvre le fichier de conf de Karma, karma.conf.js et on y ajoute ce qui suit: 

```javascript
thresholds: {
    statements: 80,
    lines: 80,
    branches: 80,
    functions: 80
  }
```

La propriété threshold est ce qui surcharge le minimum de code coverage à 80%.



## Service Tests

Les services sont souvent les fichiers les plus simples à tester. Nous allons prendre des exemples de tests unitaires synchrones et asynchrones de ValueService.

``` js
// Straight Jasmine testing without Angular's testing support
describe('ValueService', () => {
  let service: ValueService;
  beforeEach(() => { service = new ValueService(); });

  it('#getValue should return real value', () => {
    expect(service.getValue()).toBe('real value');
  });

  it('#getObservableValue should return value from observable',
    (done: DoneFn) => {
    service.getObservableValue().subscribe(value => {
      expect(value).toBe('observable value');
      done();
    });
  });

  it('#getPromiseValue should return value from a promise',
    (done: DoneFn) => {
    service.getPromiseValue().then(value => {
      expect(value).toBe('promise value');
      done();
    });
  });
});
```



### Services with dependencies

Les services dépendent souvent d'autres services qu'Angular injecte dans le constructor. Très souvent, il est simple de créer et injecter ces dépendances à la main en appelant le constructor du service.

Exemple ici un MasterService: 

```js
@Injectable()
export class MasterService {
  constructor(private valueService: ValueService) { }
  getValue() { return this.valueService.getValue(); }
}
```

En résumé, ici, nous utlisons l'unique méthode de MasterService, getValuse sur le service ValueService.

Il y a plusieurs manière de tester tout ceci: 

```js
describe('MasterService without Angular testing support', () => {
  let masterService: MasterService;

  it('#getValue should return real value from the real service', () => {
    masterService = new MasterService(new ValueService());
    expect(masterService.getValue()).toBe('real value');
  });

  it('#getValue should return faked value from a fakeService', () => {
    masterService = new MasterService(new FakeValueService());
    expect(masterService.getValue()).toBe('faked service value');
  });

  it('#getValue should return faked value from a fake object', () => {
    const fake =  { getValue: () => 'fake value' };
    masterService = new MasterService(fake as ValueService);
    expect(masterService.getValue()).toBe('fake value');
  });

  it('#getValue should return stubbed value from a spy', () => {
    // create `getValue` spy on an object representing the ValueService
    const valueServiceSpy =
      jasmine.createSpyObj('ValueService', ['getValue']);

    // set the value to return when the `getValue` spy is called.
    const stubValue = 'stub value';
    valueServiceSpy.getValue.and.returnValue(stubValue);

    masterService = new MasterService(valueServiceSpy);

    expect(masterService.getValue())
      .toBe(stubValue, 'service returned stub value');
    expect(valueServiceSpy.getValue.calls.count())
      .toBe(1, 'spy method was called once');
    expect(valueServiceSpy.getValue.calls.mostRecent().returnValue)
      .toBe(stubValue);
  });
});
```

Le premier test crée un ValueService grâce à new et le passe au constructor du MasterService.

Toutefois, tester les services dependants de cette manière est un peu compliquée et fonctionne rarement. Il vaut mieux utiliser l'utilitaire de tests Angular avec TestBed.

### Testing services with the TestBed

L'app passe par [Angular dependency Injection](https://angular.io/guide/dependency-injection) (**DI**) pour créer des services. Lorsq'un service a un service dépendant,  DI va le trouver/créer. Et si ce service possède un service dépendant, il fera de même pour celui-ci.

Il faut donc penser au premier niveau de dépendances de services mais sur le principe, on peut tout à fait laisser Di se charger de tout seul.



### Angular TestBed

_TestBed_ est l'utilitaire de test Angular le plus important. il crée un module de test Angular contruit dynamiquement qui emule un _@NgModule_.

La méthode _TestBed.configureTestingModule()_ contient un objet metadata qui possède la majorité des propriétés d'un _@NgModule_.

Pour tester un service, on rempli le _providers_ avec un tableau de services à tester.

```js
let service: ValueService;

beforeEach(() => {
  TestBed.configureTestingModule({ providers: [ValueService] });
});
```

Puis on injecte un test en appelant _TestBed.get()_ avec la classe de service comme argument.

```js
it('should use ValueService', () => {
  service = TestBed.get(ValueService);
  expect(service.getValue()).toBe('real value');
});
```

Ou dans le _beforeEach()_ si on préfère injecter le service comme faisant partie du setup

```js
beforeEach(() => {
  TestBed.configureTestingModule({ providers: [ValueService] });
  service = TestBed.get(ValueService);
});
```

 

Lorsqu'on teste un service avec dépendance, il faut fournir le mock au tableau de _providers_

Dans l'ex suivant, le mock est un [spy object](https://jasmine.github.io/2.0/introduction.html#section-Spies)  

> ## Spies
>
> Jasmine has test double functions called spies. A spy can stub any  function and tracks calls to it and all arguments. A spy only exists in  the `describe` or `it` block in which it is defined, and will be removed after each spec. There are special matchers for interacting with spies. **This syntax has changed for Jasmine 2.0.** The `toHaveBeenCalled` matcher will return true if the spy was called. The `toHaveBeenCalledWith` matcher will return true if the argument list matches any of the recorded calls to the spy.

```js
let masterService: MasterService;
let valueServiceSpy: jasmine.SpyObj<ValueService>;

beforeEach(() => {
  const spy = jasmine.createSpyObj('ValueService', ['getValue']);

  TestBed.configureTestingModule({
    // Provide both the service-to-test and its (spy) dependency
    providers: [
      MasterService,
      { provide: ValueService, useValue: spy }
    ]
  });
  // Inject both the service-to-test and its (spy) dependency
  masterService = TestBed.get(MasterService);
  valueServiceSpy = TestBed.get(ValueService);
});
```

```js
it('#getValue should return stubbed value from a spy', () => {
  const stubValue = 'stub value';
  valueServiceSpy.getValue.and.returnValue(stubValue);

  expect(masterService.getValue())
    .toBe(stubValue, 'service returned stub value');
  expect(valueServiceSpy.getValue.calls.count())
    .toBe(1, 'spy method was called once');
  expect(valueServiceSpy.getValue.calls.mostRecent().returnValue)
    .toBe(stubValue);
});
```

### Testing HTTP Services

Les Data services faisant des requêtes HTTP sur des serveurs distants injectent l'Angular HttpClient service et le laisse se charger des requêtes.

On peut tester un data service avec un spy `HttpClient` injecté  comme on testerait n'importe quel service avec dépendance.

```js
let httpClientSpy: { get: jasmine.Spy };
let heroService: HeroService;

beforeEach(() => {
  // TODO: spy on other methods too
  httpClientSpy = jasmine.createSpyObj('HttpClient', ['get']);
  heroService = new HeroService(<any> httpClientSpy);
});

it('should return expected heroes (HttpClient called once)', () => {
  const expectedHeroes: Hero[] =
    [{ id: 1, name: 'A' }, { id: 2, name: 'B' }];

  httpClientSpy.get.and.returnValue(asyncData(expectedHeroes));

  heroService.getHeroes().subscribe(
    heroes => expect(heroes).toEqual(expectedHeroes, 'expected heroes'),
    fail
  );
  expect(httpClientSpy.get.calls.count()).toBe(1, 'one call');
});

it('should return an error when the server returns a 404', () => {
  const errorResponse = new HttpErrorResponse({
    error: 'test 404 error',
    status: 404, statusText: 'Not Found'
  });

  httpClientSpy.get.and.returnValue(asyncError(errorResponse));

  heroService.getHeroes().subscribe(
    heroes => fail('expected an error, not heroes'),
    error  => expect(error.message).toContain('test 404 error')
  );
});
```

> The `HeroService` methods return `Observables`. You must *subscribe* to an observable to (a) cause it to execute and (b) assert that the method succeeds or fails.
>
> The `subscribe()` method takes a success (`next`) and fail (`error`) callback. Make sure you provide *both* callbacks so that you capture errors. Neglecting to do so produces an asynchronous uncaught observable error that the test runner will likely attribute to a completely different test.



#### HttpClientTestingModule

Etendre les interactions entre data service et `HttpClient` peut s'avérer difficile avec des spies. Ce types de tests se font plus facilement avec le `HttpClientTestingModule`.



## Component Test Basics



### Component class testing

Le test de component se fait tout comme on peut tester un service. 

Prenons en ex ce LightSwitchComponent qui toggle une lumière sur on ou off lorsqu'on clique sur le bouton.

```js
@Component({
  selector: 'lightswitch-comp',
  template: `
    <button (click)="clicked()">Click me!</button>
    <span>{{message}}</span>`
})
export class LightswitchComponent {
  isOn = false;
  clicked() { this.isOn = !this.isOn; }
  get message() { return `The light is ${this.isOn ? 'On' : 'Off'}`; }
}
```

On peut vouloir tester qu'au `clicked()` le passage à on/on laisse le bon message.

Cette classe component n'a pas de dépendance. Pour tester un service sans dépendance  on crée avec `new()`, on tape sur son API et on révupère les exceptions. Eh bien, on fera de même avec le component class. 

```js
describe('LightswitchComp', () => {
  it('#clicked() should toggle #isOn', () => {
    const comp = new LightswitchComponent();
    expect(comp.isOn).toBe(false, 'off at first');
    comp.clicked();
    expect(comp.isOn).toBe(true, 'on after click');
    comp.clicked();
    expect(comp.isOn).toBe(false, 'off after second click');
  });

  it('#clicked() should set #message to "is on"', () => {
    const comp = new LightswitchComponent();
    expect(comp.message).toMatch(/is off/i, 'off at first');
    comp.clicked();
    expect(comp.message).toMatch(/is on/i, 'on after clicked');
  });
});
```

Lorsqu’un component a des dépendances, il est préférable d'utiliser le `TestBed` pour créer le component et ses dépendances. 

ex avec ce `WelcomeCompnent` qui a dépend de `userService` pour connaître le nom des user à saluer. 

```js
export class WelcomeComponent  implements OnInit {
  welcome: string;
  constructor(private userService: UserService) { }

  ngOnInit(): void {
    this.welcome = this.userService.isLoggedIn ?
      'Welcome, ' + this.userService.user.name : 'Please log in.';
  }
}
```

On commencerait par créer le mock du `userService`

```js
class MockUserService {
  isLoggedIn = true;
  user = { name: 'Test User'};
};
```

Puis, on injecte le component **ET** le service dans la config `TestBed`

```js
beforeEach(() => {
  TestBed.configureTestingModule({
    // provide the component-under-test and dependent service
    providers: [
      WelcomeComponent,
      { provide: UserService, useClass: MockUserService }
    ]
  });
  // inject both the component and the dependent service.
  comp = TestBed.get(WelcomeComponent);
  userService = TestBed.get(UserService);
});
```

Enfin,  on teste le component class .

```js
it('should not have welcome message after construction', () => {
  expect(comp.welcome).toBeUndefined();
});

it('should welcome logged in user after Angular calls ngOnInit', () => {
  comp.ngOnInit();
  expect(comp.welcome).toContain(userService.user.name);
});

it('should ask user to log in if not logged in after ngOnInit', () => {
  userService.isLoggedIn = false;
  comp.ngOnInit();
  expect(comp.welcome).not.toContain(userService.user.name);
  expect(comp.welcome).toContain('log in');
});
```



### Component DOM testing

Un component interagi avec le DOM et d'autres components. Les tests _class-only_ peuvent uniquement gérer les problèmes de classe mais pas si le component va régir comme il le devrait, répondre correctement aux usages du user...

Aucun des tests _class-only_ vus plus haut ne peuvent répondre à ce genre de questions: 

* Est-ce que `LightSwitch.clicked()` est lié à quelque chose que le user peut appeler ou utiliser?
* Est-ce que le `LightSwitch.message` est affiché?

etc...

Pour répondre à ce genre de questions, il faut créer les éléments DOM associés aux components, on doit examiner le DOM pour confirmer qu'ils sont correctement affichés au bon moment et il faut simuler les interactions user avec l'ecran pour déterminer à quel moment ces interactions peuvent poser problème.



#### CLI-generated tests

Le CLI crée un fichier test initial par défaut lorsqu'on demande de générer un nouveau component .

ex de fichier de test: 

```js
import { async, ComponentFixture, TestBed } from '@angular/core/testing';
import { BannerComponent } from './banner.component';

describe('BannerComponent', () => {
  let component: BannerComponent;
  let fixture: ComponentFixture<BannerComponent>;

  beforeEach(async(() => {
    TestBed.configureTestingModule({
      declarations: [ BannerComponent ]
    })
    .compileComponents();
  }));

  beforeEach(() => {
    fixture = TestBed.createComponent(BannerComponent);
    component = fixture.componentInstance;
    fixture.detectChanges();
  });

  it('should create', () => {
    expect(component).toBeDefined();
  });
});
```

#### CreateComponent()

A près avoir configuré TestBed, on va s'intérésser à `createComponent()`. 

```js
const fixture = TestBed.createComponent(BannerComponent);
```

Ici, `TestBed.createComponent()` crée une instance de `BannerComponent` et ajoute un élément similaire au DOM test-runner et retourne un `ComponentFixture`.

#### ComponentFixture

Le `ComponentFixture` est un test auto qui permet d'intérragir avec le component créé et son élémént correspondant.

On peut donc accéder à l'instance de component à travers la fixture et confirmer qu'elle existe avec Jasmine.



```js
const component = fixture.componentInstance;
expect(component).toBeDefined();
```



#### beforeEach()

Plus notre component evoluera plus on ajoutera de tests. Plutôt que dupliquer la conf TesBed pour chaque test, on va refactoriser pour inclure la conf dans un `beforEach( )` Jasmine. 

```js
describe('BannerComponent (with beforeEach)', () => {
  let component: BannerComponent;
  let fixture: ComponentFixture<BannerComponent>;

  beforeEach(() => {
    TestBed.configureTestingModule({
      declarations: [ BannerComponent ]
    });
    fixture = TestBed.createComponent(BannerComponent);
    component = fixture.componentInstance;
  });

  it('should create', () => {
    expect(component).toBeDefined();
  });
});
```

Maintenant, nous allons ajouter un test qui récupère l'élément du component avec `fixture.nativeElement` et vérifier que le texte attendu ressors bien. 

```js
it('should contain "banner works!"', () => {
  const bannerElement: HTMLElement = fixture.nativeElement;
  expect(bannerElement.textContent).toContain('banner works!');
});
```



#### nativeElement

La valeur de `componentFixture.nativeElement` est de type `any`. 

> Angular ne peut pas savoir au moment de la compilation quel type d’élément HTML est nativeElement ou s’il s’agit même d’un élément HTML. L'application peut s'exécuter sur une plateforme autre que le navigateur, telle que le serveur ou un Web Worker, où l'élément peut avoir une API réduite ou ne pas exister du tout.
>
> Les tests de ce guide sont conçus pour s'exécuter dans un navigateur. Ainsi, une valeur nativeElement sera toujours un HTMLElement ou l'une de ses classes dérivées.

Sachant que nous avons à faire à un HTMLElement on peut utiliser le standard HTML `querySelector` pour mieux naviguer dans l'arborescence.

Voici un autre test  qui appelle `HTMLElement.querySelector` pour récupérer le paragraphe et checker le texte du banner:

```js
it('should have <p> with "banner works!"', () => {
  const bannerElement: HTMLElement = fixture.nativeElement;
  const p = bannerElement.querySelector('p');
  expect(p.textContent).toEqual('banner works!');
});
```



#### DebugElement

La _fixture_ Angular alimente l'élément du component directement depuis `fixture.nativeElement`

```js
const bannerElement: HTMLElement = fixture.nativeElement;
```

Il s'agit d'une méthode tout à fait pratique, implémentée comme `fixture.debugElement.nativeElement`.

```js
const bannerDe: DebugElement = fixture.debugElement;
const bannerEl: HTMLElement = bannerDe.nativeElement;
```

> Il y a une raison à ce chemin détourner pour accéder à l'élément en question.
>
> Les propriétés de nativeElement dépendent de l'environnement d'exécution. Nous pourrions exécuter ces tests sur une plate-forme sans navigateur qui ne possède pas de DOM ou dont l’émulation DOM ne prend pas en charge l’API HTMLElement complète.
>
> Angular s'appuie sur l'abstraction DebugElement pour fonctionner en toute sécurité sur toutes les plateformes prises en charge. Au lieu de créer une arborescence d'éléments HTML,Angular crée une arborescence DebugElement qui englobe les éléments natifs de la plate-forme d'exécution. La propriété nativeElement décompresse DebugElement et renvoie l'objet élément spécifique à la plate-forme.
>
> Étant donné que les exemples de tests de ce guide sont conçus pour s'exécuter uniquement dans un navigateur, un élément nativeElement dans ces tests est toujours un élément HTMLElement dont vous pouvez explorer les méthodes et propriétés connues au sein d'un test.

Here's the previous test, re-implemented with `fixture.debugElement.nativeElement`:

```js
it('should find the <p> with fixture.debugElement.nativeElement)', () => {
  const bannerDe: DebugElement = fixture.debugElement;
  const bannerEl: HTMLElement = bannerDe.nativeElement;
  const p = bannerEl.querySelector('p');
  expect(p.textContent).toEqual('banner works!');
});
```

`DebugElement` a d'autres méthodes et propriétés utiles pour les tests. 

Pour utliser le `DebugElement` il faut l'importer de la librairie core d'Angular.

```js
import { DebugElement } from '@angular/core';
```

#### By.css()

Bien que les tests de ce guide s’exécutent tous dans le navigateur, certaines applications peuvent au moins être exécutées sur une plate-forme différente.

Par exemple, le component peut d'abord apparaître sur le serveur dans le cadre d'une stratégie visant à accélérer le lancement de l'application sur des périphériques mal connectés. Le moteur de rendu côté serveur peut ne pas prendre en charge l'intégralité de l'API d'élément HTML. S'il ne prend pas en charge `querySelector`, le test précédent peut échouer.

`DebugElement` propose des méthodes de requête qui fonctionnent pour toutes les plates-formes prises en charge. Ces méthodes de requête utilisent une fonction de prédicat qui renvoie true lorsqu'un nœud de l'arborescence `DebugElement` correspond aux critères de sélection.

On crée un _predicate_ à l'aide de la classe By importée depuis la librairie de la plateforme. 

```js
import { By } from '@angular/platform-browser';
```

L'ex suivant réimplémente le test précédent avec `DebugElement.query()` et le browser avec `By.css()`.

```js
it('should find the <p> with fixture.debugElement.query(By.css)', () => {
  const bannerDe: DebugElement = fixture.debugElement;
  const paragraphDe = bannerDe.query(By.css('p'));
  const p: HTMLElement = paragraphDe.nativeElement;
  expect(p.textContent).toEqual('banner works!');
});
```

* La méthode statique `By.css ()` sélectionne les nœuds `DebugElement` avec un sélecteur CSS standard.
* Le `query` retrourne un `DebugElement` pour le paragraphe.

>  Lorsqu'on filtre avec un CSS selector et qu'on ne teste que les propriétés d'un `nativeElement`, le `By.css` est peut-être exagéré.
>
> Il est souvent plus clean et simple de filtrer avec une méthode HTMLElement standard  comme un `querySelector()` ou un `querySelectorAll()`.

​     

## Component Test Scenarios 

Nous allonns ici explorer une bonne partie des scenarios de tests possibles.



#### Component binding

Prenons le Component `BannerComponent` et ajoutons lui un titre dynamique. 

```js
@Component({
  selector: 'app-banner',
  template: '<h1>{{title}}</h1>',
  styles: ['h1 { color: green; font-size: 350%}']
})
export class BannerComponent {
  title = 'Test Tour of Heroes';
}
```

Nous souhaitons tout d'abord ajouter un test qui confirmera que le component affiche bien le bon contenu comme il le devrait.

##### Query for the \<h1>

Nous allons écrire uné séquence de tests qui inspecte la valeur du \<h1> qui englobe la l'interpolation binding de la propriété _titre_.

On met à jour le  `beforEach` pour trouver l'élément avec un `querySelector` et l'assigner à la variable h1.

```js
let component: BannerComponent;
let fixture:   ComponentFixture<BannerComponent>;
let h1:        HTMLElement;

beforeEach(() => {
  TestBed.configureTestingModule({
    declarations: [ BannerComponent ],
  });
  fixture = TestBed.createComponent(BannerComponent);
  component = fixture.componentInstance; // BannerComponent test instance
  h1 = fixture.nativeElement.querySelector('h1');
});
```

##### createCompent() does not bind data

Pour notre premier test, nous cherchons à vérifier que l'ecran affiche le _title_ par defaut. Instinctivement on voudrait l'écrire comme ceci: 

```js
it('should display original title', () => {
  expect(h1.textContent).toContain(component.title);
});
```

Mais le test fail avec ce message: 

```js
expected '' to contain 'Test Tour of Heroes'.
```

> Le binding se produit lorsqu'Angular detecte un changement.
>
> En production, la détection des modifications se déclenche automatiquement lorsque Angular crée un component ou que l'utilisateur entre une activité asynchrone (par exemple, AJAX).

Le `TestBed.createComponent` ne déclenche pas la détection de changement. On peut le confirmer dans le test revu:

```js
it('no title in the DOM after createComponent()', () => {
  expect(h1.textContent).toEqual('');
});
```

##### detectChanges()

Si nous demandons à  `TestBed` d'effectuer le data binding en appelant `fixtures.changes()`alors le \<h1> aura le _title_ attendu.

```js
it('should display original title after detectChanges()', () => {
  fixture.detectChanges();
  expect(h1.textContent).toContain(component.title);
});
```

> La détection de changement retardé est intentionnelle et utile. Il donne au testeur l'occasion d'inspecter et de modifier l'état du composant avant qu'Angular initie la liaison de données et appelle les hooks de cycle de vie.

Voici un autre test qui modifie la propriété _title_ du component avant d'appeler `fixture.detectChanges ()`.

```js
it('should display a different test title', () => {
  component.title = 'Test Title';
  fixture.detectChanges();
  expect(h1.textContent).toContain('Test Title');
});
```



##### Change an input with dispatchEvent()

Pour simuler une entrée utilisateur, on peut rechercher l'élément input et définir sa propriété value.

On appelle `fixture.detectChanges()` pour déclencher la détection des modifications par Angular. Mais il y a une étape essentielle et intermédiaire.

Angular ne sait pas qu'on définie la propriété _value_ de l'élément en entrée. Il ne lira pas cette propriété tant qu'on n'aura pas déclenché l'événement input de l'élément en appelant `dispatchEvent()`. Ensuite, on appelle `detectChanges ()`.

```js
it('should convert hero name to Title Case', () => {
  // get the name's input and display elements from the DOM
  const hostElement = fixture.nativeElement;
  const nameInput: HTMLInputElement = hostElement.querySelector('input');
  const nameDisplay: HTMLElement = hostElement.querySelector('span');

  // simulate user entering a new name into the input box
  nameInput.value = 'quick BROWN  fOx';

  // dispatch a DOM event so that Angular learns of input value change.
  nameInput.dispatchEvent(newEvent('input'));

  // Tell Angular to update the display binding through the title pipe
  fixture.detectChanges();

  expect(nameDisplay.textContent).toBe('Quick Brown  Fox');
});
```



#### Component with external files 

Notre `BannerComponent` plus haut est définie avec un _inline template_ et un _inline css_, spécifiés dans les propritétés respectives de `@Component.template` et `@Component.styles`.

De nombreux components spécifient des modèles externes et des css externes avec les propriétés `@ Component.templateUrl` et `@ Component.styleUrls` respectivement, comme le fait la variante suivante de `BannerComponent`.

```js
@Component({
  selector: 'app-banner',
  templateUrl: './banner-external.component.html',
  styleUrls:  ['./banner-external.component.css']
})
```

>  Cette syntaxe demande au compileur d'Angular de lire le fichier externe pendant la compilation du component.

Ce n'est pas un problème lorsqu'on exécute la commande CLI ng test car elle compile l'application avant d'exécuter les tests.

Toutefois, si on éxécute les tests dans un environnement autre que l'interface de ligne de commande, les tests de ce component peuvent échouer. Par exemple, si éxécute les tests `BannerComponent` dans un environnement de codage Web tel que [plunker](https://plnkr.co/), un message comme celui-ci s'affiche:

```powershell
Error: This test module uses the component BannerComponent
which is using a "templateUrl" or "styleUrls", but they were never compiled.
Please call "TestBed.compileComponents" before your test.
```

Pour corriger ce problème, il sddit d'utiliser `compileComponents()` comme précisé dans le message d'erreur.



### Component with a dependency 

Les components ont souvent des dépendances de service.

`WelcomeComponent` affiche un message de bienvenue à l'utilisateur connecté. Il sait qui est l'utilisateur est basé sur une propriété du `UserService` injecté:

```js
import { Component, OnInit } from '@angular/core';
import { UserService }       from '../model/user.service';

@Component({
  selector: 'app-welcome',
  template: '<h3 class="welcome"><i>{{welcome}}</i></h3>'
})
export class WelcomeComponent  implements OnInit {
  welcome: string;
  constructor(private userService: UserService) { }

  ngOnInit(): void {
    this.welcome = this.userService.isLoggedIn ?
      'Welcome, ' + this.userService.user.name : 'Please log in.';
  }
}
```

`WelcomeComponent` a une logique de décision qui interagit avec le service, une logique qui rend ce composant intéressant à tester. Voici la configuration du module de test pour le fichier de spécifications `app / welcome / welcome.component.spec.ts`:

```js
TestBed.configureTestingModule({
   declarations: [ WelcomeComponent ],
// providers:    [ UserService ]  // NO! Don't provide the real service!
                                  // Provide a test-double instead
   providers:    [ {provide: UserService, useValue: userServiceStub } ]
});
```

Cette fois, en plus de déclarer le composant à tester, la configuration ajoute un provider `UserService` à la liste des providers. Mais pas le vrai `UserService`.



##### Provider service test doubles

Un component à tester ne doit pas nécessairement être injecté avec de vrais services. En fait, il est généralement préférable qu’il s’agisse de tests en double (stubs, fakes, spies, or mocks). Le but de la spécification est de tester le component, pas le service, et les services réels peuvent poser problème.

Injecter le véritable `UserService` pourrait être un cauchemar. Le service réel peut demander à l'utilisateur des informations d'identification de connexion et tenter d'atteindre un serveur d'authentification. Ces comportements peuvent être difficiles à intercepter. Il est beaucoup plus facile et sûr de créer et d’enregistrer un double test à la place du véritable `UserService`.

Cette suite de tests particulière fournit une maquette minimale de `UserService` qui répond aux besoins de `WelcomeComponent` et de ses tests:

```js
let userServiceStub: Partial<UserService>;

userServiceStub = {
  isLoggedIn: true,
  user: { name: 'Test User'}
};
```



##### Get Injected services

Les tests doivent avoir accès au `UserService` (stub) injecté dans `WelcomeComponent`.

Angular a un système d'injection hiérarchique. Il peut y avoir des injecteurs à plusieurs niveaux, de l'injecteur root créé par le `TestBed` à l'arborescence des components.

Le moyen le plus sûr d'obtenir le service injecté, la méthode qui fonctionne toujours, consiste à l'obtenir à partir de l'injecteur du component à tester. L'injecteur de component est une propriété de `DebugElement` du fixture.

```js
// UserService actually injected into the component
userService = fixture.debugElement.injector.get(UserService);
```



##### TestBed.get()

On peut aussi obtenir le service auprès de l'injecteur root via `TestBed.get ()`. C’est plus facile à retenir et moins verbeux. Mais cela ne fonctionne que lorsque Angular injecte le component avec l'instance de service dans l'injecteur root du test.

Dans cette suite de tests, le seul provider de `UserService` est le module de test root. Il est donc prudent d'appeler `TestBed.get()` comme suit:

```js
// UserService from the root injector
userService = TestBed.get(UserService);
```

##### Always get the service from an injector

Ne pas faire référence à l'objet `userServiceStub` fourni au module de test dans le corps de votre test. Ça ne marche pas! L'instance `userService` injectée dans le component est un objet complètement différent, un clone du `userServiceStub` fourni.

```js
it('stub object and injected UserService should not be the same', () => {
  expect(userServiceStub === userService).toBe(false);

  // Changing the stub object has no effect on the injected service
  userServiceStub.isLoggedIn = false;
  expect(userService.isLoggedIn).toBe(true);
});
```

##### Final setup and tests 

Voici le `beforeEach()` complet, utilisant `TestBed.get()`:

```js
let userServiceStub: Partial<UserService>;

beforeEach(() => {
  // stub UserService for test purposes
  userServiceStub = {
    isLoggedIn: true,
    user: { name: 'Test User'}
  };

  TestBed.configureTestingModule({
     declarations: [ WelcomeComponent ],
     providers:    [ {provide: UserService, useValue: userServiceStub } ]
  });

  fixture = TestBed.createComponent(WelcomeComponent);
  comp    = fixture.componentInstance;

  // UserService from the root injector
  userService = TestBed.get(UserService);

  //  get the "welcome" element by CSS selector (e.g., by class name)
  el = fixture.nativeElement.querySelector('.welcome');
});
```



Et quelques tests: 

```js
it('should welcome the user', () => {
  fixture.detectChanges();
  const content = el.textContent;
  expect(content).toContain('Welcome', '"Welcome ..."');
  expect(content).toContain('Test User', 'expected name');
});

it('should welcome "Bubba"', () => {
  userService.user.name = 'Bubba'; // welcome message hasn't been shown yet
  fixture.detectChanges();
  expect(el.textContent).toContain('Bubba');
});

it('should request login if not logged in', () => {
  userService.isLoggedIn = false; // welcome message hasn't been shown yet
  fixture.detectChanges();
  const content = el.textContent;
  expect(content).not.toContain('Welcome', 'not welcomed');
  expect(content).toMatch(/log in/i, '"log in"');
});
```

Le premier est un sanity test ; cela confirme que le `UserService stubbed` est appelé et fonctionne.

Les tests restants confirment la logique du component lorsque le service renvoie des valeurs différentes. Le deuxième test valide l'effet de la modification du nom d'utilisateur. Le troisième test vérifie que le component affiche le message approprié lorsqu'il n'y a aucun utilisateur connecté.



### Components with Async service

Dans cet exemple, le modèle `AboutComponent` héberge un `TwainComponent`. Le `TwainComponent` affiche les citations de Mark Twain.

```js
template: `
  <p class="twain"><i>{{quote | async}}</i></p>
  <button (click)="getQuote()">Next quote</button>
  <p class="error" *ngIf="errorMessage">{{ errorMessage }}</p>`,
```

Notons que la valeur de la propriété _quote_ du component passe par un AsyncPipe. Cela signifie que la propriété renvoie soit une promesse, soit un observable.

Dans cet exemple, la méthode `TwainComponent.getQuote ()` vous indique que la propriété quote renvoie un observable.

```js
getQuote() {
  this.errorMessage = '';
  this.quote = this.twainService.getQuote().pipe(
    startWith('...'),
    catchError( (err: any) => {
      // Wait a turn because errorMessage already set once this turn
      setTimeout(() => this.errorMessage = err.message || err.toString());
      return of('...'); // reset message to placeholder
    })
  );
```

`TwainComponent` obtient des citations d'un `TwainService` injecté. Le component démarre l'observable renvoyé avec une valeur de substitution ('...') avant que le service ne puisse renvoyer son premier devis.



`CatchError` intercepte les erreurs de service, prépare un message d'erreur et renvoie la placeholder value. Il faut attendre une coche pour définir le message errorMessage afin d'éviter de mettre à jour ce message deux fois au cours du même cycle de détection de changement.

Ce sont toutes les fonctionnalités que nous pourrons tester.



##### Testing with a spy

Lors du test d'un component, seule l'API publique du service doit compter. En général, les tests eux-mêmes ne doivent pas appeler des serveurs distants. Ils devraient imiter de tels appels. La configuration de cette `app / twain / twain.component.spec.ts` montre une façon de le faire:

```js
beforeEach(() => {
  testQuote = 'Test Quote';

  // Create a fake TwainService object with a `getQuote()` spy
  const twainService = jasmine.createSpyObj('TwainService', ['getQuote']);
  // Make the spy return a synchronous Observable with the test data
  getQuoteSpy = twainService.getQuote.and.returnValue( of(testQuote) );

  TestBed.configureTestingModule({
    declarations: [ TwainComponent ],
    providers:    [
      { provide: TwainService, useValue: twainService }
    ]
  });

  fixture = TestBed.createComponent(TwainComponent);
  component = fixture.componentInstance;
  quoteEl = fixture.nativeElement.querySelector('.twain');
});
```

Focus sur le spy: 

```js
// Create a fake TwainService object with a `getQuote()` spy
const twainService = jasmine.createSpyObj('TwainService', ['getQuote']);
// Make the spy return a synchronous Observable with the test data
getQuoteSpy = twainService.getQuote.and.returnValue( of(testQuote) );
```

Le spy est conçu pour que tout appel à `getQuote` reçoive un Observable avec un devis de test. Contrairement à la méthode réelle `getQuote ()`, ce spy contourne le serveur et renvoie un Observable synchrone dont la valeur est disponible immédiatement.

On peut écrire de nombreux tests utiles avec ce spy, même si son Observable est synchrone.



##### Synchronous tests

Un avantage clé d'un Observable synchrone est qu'on peut souvent transformer des processus asynchrones en tests synchrones.

```js
it('should show quote after component initialized', () => {
  fixture.detectChanges(); // onInit()

  // sync spy result shows testQuote immediately after init
  expect(quoteEl.textContent).toBe(testQuote);
  expect(getQuoteSpy.calls.any()).toBe(true, 'getQuote called');
});
```

Étant donné que le résultat du spy est renvoyé de manière synchrone, la méthode getQuote () met à jour le message à l'écran immédiatement après le premier cycle de détection de modification au cours duquel Angular appelle ngOnInit.

Bien que le spy service renvoie une erreur de manière synchrone, la méthode du component appelle `setTimeout ()`. Le test doit attendre au moins un tour complet du moteur JavaScript avant que la valeur ne soit disponible. Le test doit devenir asynchrone.

##### Async test with fakeAsync()

Pour utiliser les fonctionnalités de `fakeAsync()` il faut importer `zone-testing`.

Le test suivant confirme le comportement attendu lorsque le service renvoie un ErrorObservable.

```js
it('should display error when TwainService fails', fakeAsync(() => {
  // tell spy to return an error observable
  getQuoteSpy.and.returnValue(
    throwError('TwainService test failure'));

  fixture.detectChanges(); // onInit()
  // sync spy errors immediately after init

  tick(); // flush the component's setTimeout()

  fixture.detectChanges(); // update errorMessage within setTimeout()

  expect(errorMessage()).toMatch(/test failure/, 'should display error');
  expect(quoteEl.textContent).toBe('...', 'should show placeholder');
}));
```

Notez que la fonction `it()` reçoit un argument de la forme suivante.

```js
fakeAsync(() => { /* test body */ })`
```

La fonction `fakeAsync ()` active un style de codage linéaire en exécutant le corps du test dans une zone de test fakeAsync spéciale. Le corps du test semble être synchrone. Il n'y a pas de syntaxe imbriquée (comme un Promise.then ()) pour perturber le flux de contrôle.

##### The *tick()* function

Vous devez appeler `tick()` pour avancer l'horloge (virtuelle).

L'appel de `tick()` simule le passage du temps jusqu'à la fin de toutes les activités asynchrones en attente. Dans ce cas, il attend `setTimeout()` du gestionnaire d'erreurs;

La fonction `tick()` accepte les millisecondes comme paramètre (la valeur par défaut est 0 si elle n’est pas fournie). Le paramètre représente l'avancée de l'horloge virtuelle. Par exemple, si on a un test setTimeout (fn, 100) dans fakeAsync (), vous devez utiliser tick (100) pour déclencher le rappel fn.

```js
it('should run timeout callback with delay after call tick with millis', fakeAsync(() => {
     let called = false;
     setTimeout(() => { called = true; }, 100);
     tick(100);
     expect(called).toBe(true);
   }));
```

La fonction `tick()` est l’un des utilitaires de test Angular importé avec `TestBed`. C'est un compagnon de `fakeAsync ()` et on ne peut l'appeler que dans un corps fakeAsync ().

##### Comparing dates inside fakeAsync()

`fakeAsync ()` simule le passage du temps, ce qui vous permet de calculer la différence entre les dates à l'intérieur de `fakeAsync ()`.

```js
it('should get Date diff correctly in fakeAsync', fakeAsync(() => {
     const start = Date.now();
     tick(100);
     const end = Date.now();
     expect(end - start).toBe(100);
   }));
```

##### jasmine.clock with fakeAsync()

Jasmine fournit également une fonction d'horloge pour imiter des dates. Angular exécute automatiquement les tests exécutés après que `jasmine.clock ()`. `Install ()` a été appelé dans une méthode `fakeAsync ()` jusqu'à ce que `jasmine.clock (). Uninstall ()` soit appelé. `fakeAsync ()` n'est pas nécessaire et génère une erreur si elle est imbriquée.

Par défaut, cette fonctionnalité est désactivée. Pour l'activer, il faut définir un indicateur global avant l'importation de la zone-testing.

Si on utilise Angular CLI, il faut configurer ce flag dans `src/test.ts`.

```js
(window as any)['__zone_symbol__fakeAsyncPatchLock'] = true;
import 'zone.js/dist/zone-testing';
```

```js
describe('use jasmine.clock()', () => {
  // need to config __zone_symbol__fakeAsyncPatchLock flag
  // before loading zone.js/dist/zone-testing
  beforeEach(() => { jasmine.clock().install(); });
  afterEach(() => { jasmine.clock().uninstall(); });
  it('should auto enter fakeAsync', () => {
    // is in fakeAsync now, don't need to call fakeAsync(testFn)
    let called = false;
    setTimeout(() => { called = true; }, 100);
    jasmine.clock().tick(100);
    expect(called).toBe(true);
  });
});
```

##### Using the RxJS scheduler inside fakeAsync()

On peut également utiliser le planificateur RxJS dans `fakeAsync ()` comme avec `setTimeout ()`ou `setInterval ()`, mais on doit importer le fichier `zone.js / dist / zone-patch-rxjs-fake-async` pour appliquer un correctif au planificateur RxJS.

```js
it('should get Date diff correctly in fakeAsync with rxjs scheduler', fakeAsync(() => {
     // need to add `import 'zone.js/dist/zone-patch-rxjs-fake-async'
     // to patch rxjs scheduler
     let result = null;
     of ('hello').pipe(delay(1000)).subscribe(v => { result = v; });
     expect(result).toBeNull();
     tick(1000);
     expect(result).toBe('hello');

     const start = new Date().getTime();
     let dateDiff = 0;
     interval(1000).pipe(take(2)).subscribe(() => dateDiff = (new Date().getTime() - start));

     tick(1000);
     expect(dateDiff).toBe(1000);
     tick(1000);
     expect(dateDiff).toBe(2000);
   }));
```

##### Async Observables

L'Observable async a été produite par un assistant `asyncData`. L'assistant `asyncData` est une fonction utilitaire que l'on doit écrire soit-même. Ou copier celui-ci à partir de l'exemple de code.

```js
/** Create async observable that emits-once and completes
 *  after a JS engine turn */
export function asyncData<T>(data: T) {
  return defer(() => Promise.resolve(data));
}
```

> L'Observable de cet assistant émet la valeur de données lors du prochain tour du moteur JavaScript.
>
> L'opérateur RxJS defer () renvoie un Observable. Il faut une fonction d'usine qui retourne une promesse ou un observable. Lorsqu'un élément s'abonne pour différer l'observable, il ajoute l'abonné à un nouvel observable créé avec cette fabrique.
>
> L'opérateur defer () transforme Promise.resolve () en une nouvelle observable qui, comme HttpClient, émet une fois et se termine. Les abonnés sont désabonnés après avoir reçu la valeur de données.
>
> Il existe une aide similaire pour générer une erreur asynchrone.
>
> ```js
> /** Create async observable error that errors
>  *  after a JS engine turn */
> export function asyncError<T>(errorObject: any) {
>   return defer(() => Promise.reject(errorObject));
> }
> ```

##### More async tests

Maintenant que le `getQuote ()` spy renvoie des observables asynchrones, la plupart de nos tests devront également être asynchrones.

Voici un test `fakeAsync ()` qui illustre le flux de données auquel nous nous attendions dans le monde réel.

```js
it('should show quote after getQuote (fakeAsync)', fakeAsync(() => {
  fixture.detectChanges(); // ngOnInit()
  expect(quoteEl.textContent).toBe('...', 'should show placeholder');

  tick(); // flush the observable to get the quote
  fixture.detectChanges(); // update view

  expect(quoteEl.textContent).toBe(testQuote, 'should show quote');
  expect(errorMessage()).toBeNull('should not show error');
}));
```

Notons que l'élément _quote_ affiche la placeholder value ('...') après `ngOnInit ()`. La première citation n'est pas encore arrivée.

Pour effacer la première citation de l'observable, on appelle `tick ()`. Ensuite, on appelle `detectChanges ()` pour indiquer à Angular de mettre à jour l'écran.

Ensuite, vous pouvez affirmer que l'élément _quote_ affiche le texte attendu.



### Component Marble tests

Les tests `TwainComponent` précédents simulaient une réponse observable asynchrone de `TwainService` avec les utilitaires `asyncData` et `asyncError`.

Ce sont des fonctions courtes et simples. Malheureusement, elles sont trop simples pour de nombreux scénarios courants. Un Observable émet souvent plusieurs fois, peut-être après un délai important. Un component peut coordonner plusieurs éléments Observables avec des séquences de valeurs et d’erreurs se chevauchant.

**RxJS marble testing** est un excellent moyen de tester des scénarios Observables, simples et complexes. Vous avez probablement vu _les diagrammes marbles_ illustrant le fonctionnement des Observables. Le marble testing utilise un langage marble similaire pour spécifier les flux observables et les attentes dans vos tests.

Les exemples suivants reprennent deux des tests `TwainComponent` avec des marbles tests.

On commence par installer le package jasmine-marbles NPM. On importe ensuite les symboles dont on aura besoin.

```js
import { cold, getTestScheduler } from 'jasmine-marbles';
```

Voici le test complet pour obtenir une citation : 

```js
it('should show quote after getQuote (marbles)', () => {
  // observable test quote value and complete(), after delay
  const q$ = cold('---x|', { x: testQuote });
  getQuoteSpy.and.returnValue( q$ );

  fixture.detectChanges(); // ngOnInit()
  expect(quoteEl.textContent).toBe('...', 'should show placeholder');

  getTestScheduler().flush(); // flush the observables

  fixture.detectChanges(); // update view

  expect(quoteEl.textContent).toBe(testQuote, 'should show quote');
  expect(errorMessage()).toBeNull('should not show error');
});
```

Notons que le test de Jasmine est synchrone. Il n'y a pas de `fakeAsync ()`. Le marble test utilise un planificateur de test pour simuler le passage du temps dans un test synchrone.

Le marble testing définit une observable à froid qui attend trois trames (---), émet une valeur (x) et complète (|). Dans le deuxième argument, vous mappez le marqueur de valeur (x) sur la valeur émise (testQuote).

```js
const q$ = cold('---x|', { x: testQuote });
```

La marble librairy construit l'observable correspondant, que le test définit comme valeur de retour pour le `getQuote` spy.

Lorsque nous sommes prêt à activer les marble observables, nous indiquons à `TestScheduler` de vider sa file d'attente de tâches préparées comme celle-ci.

```js
getTestScheduler().flush(); // flush the observables
```

Cette étape a un objectif analogue à `tick ()` et à `whenStable ()` dans les exemples précédents `fakeAsync ()` et `async ()`. Le reste du test est le même que ces exemples.



##### Marble error testing

Voici la version marble testing du test d'erreur getQuote ().

```js
it('should display error when TwainService fails', fakeAsync(() => {
  // observable error after delay
  const q$ = cold('---#|', null, new Error('TwainService test failure'));
  getQuoteSpy.and.returnValue( q$ );

  fixture.detectChanges(); // ngOnInit()
  expect(quoteEl.textContent).toBe('...', 'should show placeholder');

  getTestScheduler().flush(); // flush the observables
  tick();                     // component shows error after a setTimeout()
  fixture.detectChanges();    // update error message

  expect(errorMessage()).toMatch(/test failure/, 'should display error');
  expect(quoteEl.textContent).toBe('...', 'should show placeholder');
}));
```

C'est toujours un test asynchrone, appelant `fakeAsync ()` et `tick ()`, car le component lui-même appelle `setTimeout ()` lors du traitement des erreurs.

Voici la définition du marble Observable

```js
const q$ = cold('---#|', null, new Error('TwainService test failure'));
```

C'est un cold Observable  qui attend trois trames, puis émet une erreur. Le dièse (#) indique la synchronisation de l'erreur spécifiée dans le troisième argument. Le second argument est nul car l'observable n'émet jamais de valeur

> ##### Learn about marble testing
>
> Une **marble frame** est une unité virtuelle de test du temps. Chaque symbole (-, x, |, #) marque le passage d’une image à l’autre.
>
> Un **Cold Observable** ne produit pas de valeurs tant que vous n'y êtes pas abonné. La plupart de vos observables d'application sont **cold**. Toutes les méthodes HttpClient renvoient des  **cold observables**.
>
> Un **Hot Observable** produit déjà des valeurs avant de vous y abonner. L'observable `Router.events`, qui rapporte l'activité du routeur, est un **hot observable**.
>
> Le **RxJS marble testing** est un sujet riche qui dépasse le cadre de ce guide. Apprenez-le sur le Web, en commençant par la [documentation officielle](https://github.com/ReactiveX/rxjs/blob/master/doc/writing-marble-tests.md).



### Component with inputs and outputs

Un component avec des _inputs_ et _outputs_ apparaît généralement dans le modèle de vue d'un component hôte. L'hôte utilise un **property binding** pour définir la propriété de _l'input_ et un **event binding** pour écouter les événements générés par la propriété de _l'output_.

L'objectif de test est de vérifier que ces _bindings_ fonctionnent comme prévu. Les tests doivent définir les valeurs d'_input_ et écouter les événements de l'_output_.

`DashboardHeroComponent` est un petit exemple de ce type de component. Il affiche un héros individuel fourni par `DashboardComponent`. En cliquant sur ce héros, il indique au `DashboardComponent`que l'utilisateur a sélectionné le héros.

Le component `DashboardHeroComponent` est intégré au modèle `DashboardComponent` de la manière suivante:

```js
<dashboard-hero *ngFor="let hero of heroes"  class="col-1-4"
  [hero]=hero  (selected)="gotoDetail($event)" >
</dashboard-hero>
```

Le component `DashboardHeroComponent` apparaît dans un `* ngFor`, qui définit la propriété d'_input_ héros de chaque composant sur la valeur de la boucle et écoute l'événement sélectionné du component.

Voici la définition complète du composant:

```js
@Component({
  selector: 'dashboard-hero',
  template: `
    <div (click)="click()" class="hero">
      {{hero.name | uppercase}}
    </div>`,
  styleUrls: [ './dashboard-hero.component.css' ]
})
export class DashboardHeroComponent {
  @Input() hero: Hero;
  @Output() selected = new EventEmitter<Hero>();
  click() { this.selected.emit(this.hero); }
}
```



##### Test DashboardHeroComponent stand-alone

Voici le fichier de spec de base: 

```js 
TestBed.configureTestingModule({
  declarations: [ DashboardHeroComponent ]
})
fixture = TestBed.createComponent(DashboardHeroComponent);
comp    = fixture.componentInstance;

// find the hero's DebugElement and element
heroDe  = fixture.debugElement.query(By.css('.hero'));
heroEl = heroDe.nativeElement;

// mock the hero supplied by the parent component
expectedHero = { id: 42, name: 'Test Name' };

// simulate the parent setting the input property with that hero
comp.hero = expectedHero;

// trigger initial data binding
fixture.detectChanges();
```

Notons que le code de configuration affecte un héros de test (attenduHero) à la propriété de héros du composant, en émulant la manière dont `DashboardComponent` le définira via le property binding dans son répéteur.

Le test suivant vérifie que le nom du héros est propagé au modèle via un binding.

```js
it('should display hero name in uppercase', () => {
  const expectedPipedName = expectedHero.name.toUpperCase();
  expect(heroEl.textContent).toContain(expectedPipedName);
});
```

Étant donné que le modèle passe le nom du héros par le biais de l'Angular `UpperCasePipe`, le test doit faire correspondre la valeur de l'élément avec le nom en majuscule.

##### Clicking

Cliquer sur le héros devrait déclencher un événement sélectionné que le component hôte (probablement `DashboardComponent`) peut entendre:

```js
it('should raise selected event when clicked (triggerEventHandler)', () => {
  let selectedHero: Hero;
  comp.selected.subscribe((hero: Hero) => selectedHero = hero);

  heroDe.triggerEventHandler('click', null);
  expect(selectedHero).toBe(expectedHero);
});
```

La propriété sélectionnée du component renvoie un `EventEmitter`, qui ressemble à un observable synchrone RxJS pour les consommateurs. Le test y est explicitement abonné, tout comme le component hôte le fait implicitement.

Si le component se comporte comme prévu, cliquer sur l'élément du héros doit indiquer à la propriété sélectionnée du component d'émettre l'objet héros.

Le test détecte cet événement via son abonnement à `selected`.

##### *triggerEventHandler*

Dans le test précédent, `heroDe` est un `DebugElement` qui représente le héros \<div>.

Il a des propriétés Angular et des méthodes qui empêchent l’interaction avec l’élément natif. Ce test appelle `DebugElement.triggerEventHandler` avec le nom d'événement "click". L'event binding "click" répond en appelant `DashboardHeroComponent.click ()`.

Angular `DebugElement.triggerEventHandler` peut déclencher tout événement lié aux données par son nom. Le deuxième paramètre est l'objet événement transmis au gestionnaire.

Le test a déclenché un événement "clic" avec un objet événement null.

```js
heroDe.triggerEventHandler('click', null);
```

Le test suppose (correctement dans ce cas) que le gestionnaire d'événements au moment de l'exécution (la méthode `click ()` du component) ne se soucie pas de l'objet event.



##### click the element

La variante de test suivante appelle la méthode `click ()` de l'élément natif, ce qui convient parfaitement à ce component.

```js
it('should raise selected event when clicked (element.click)', () => {
  let selectedHero: Hero;
  comp.selected.subscribe((hero: Hero) => selectedHero = hero);

  heroEl.click();
  expect(selectedHero).toBe(expectedHero);
});
```



##### click()* helper

Cliquer sur un bouton, une ancre ou un élément HTML arbitraire est une tâche de test courante.

Rendre cela cohérent et facile en encapsulant le processus de déclenchement de clic dans un assistant tel que la fonction `click ()` ci-dessous:

```js
/** Button events to pass to `DebugElement.triggerEventHandler` for RouterLink event handler */
export const ButtonClickEvents = {
   left:  { button: 0 },
   right: { button: 2 }
};

/** Simulate element click. Defaults to mouse left-button click event. */
export function click(el: DebugElement | HTMLElement, eventObj: any = ButtonClickEvents.left): void {
  if (el instanceof HTMLElement) {
    el.click();
  } else {
    el.triggerEventHandler('click', eventObj);
  }
}
```

Le premier paramètre est l'élément-à-cliquer. Si on le souhaite, on peut passer un objet event personnalisé en tant que second paramètre. La valeur par défaut est un objet événement de souris gauche (partiel) accepté par de nombreux gestionnaires, y compris la directive `RouterLink`.

Voici le test précédent, réécrit à l'aide du _click Helper_.

```js
it('should raise selected event when clicked (click helper)', () => {
  let selectedHero: Hero;
  comp.selected.subscribe(hero => selectedHero = hero);

  click(heroDe); // click helper with DebugElement
  click(heroEl); // click helper with native element

  expect(selectedHero).toBe(expectedHero);
});
```



### Component inside a test host

Les tests précédents ont joué le rôle de l'hôte `DashboardComponent`. Mais le component `DashboardHeroComponent` fonctionne-t-il correctement lorsqu'il est correctement lié aux données d'un component hôte?

On peut tester avec le component `DashboardComponent`. Mais cela pourrait nécessiter beaucoup d’installation, en particulier lorsque son modèle comporte un répéteur * ngFor, d’autres composants, une présentation HTML, des liaisons supplémentaires, un constructeur qui injecte plusieurs services et qu’il commence immédiatement à interagir avec ces services.

Imaginez l’effort de désactiver ces distractions, juste pour prouver un point qui peut être fait de manière satisfaisante avec un hôte de test comme celui-ci:

```js
@Component({
  template: `
    <dashboard-hero
      [hero]="hero" (selected)="onSelected($event)">
    </dashboard-hero>`
})
class TestHostComponent {
  hero: Hero = {id: 42, name: 'Test Name' };
  selectedHero: Hero;
  onSelected(hero: Hero) { this.selectedHero = hero; }
}
```

Cet hôte de test se lie à `DashboardHeroComponent` comme le ferait `DashboardComponent` mais sans le bruit du `Router`, du `HeroService` ou du répéteur * ngFor.

L'hôte de test définit la propriété d'input du héros du component avec son héros de test. Il lie l'event sélectionné du component à son gestionnaire `onSelected`, qui enregistre le héros émis dans sa propriété `selectedHero`.

Plus tard, les tests pourront facilement vérifier `selectedHero` pour vérifier que l'eventt `DashboardHeroComponent.selected` a émis le héros attendu.

La configuration des tests de l'hôte de test est similaire à celle des tests autonomes:

```js
TestBed.configureTestingModule({
  declarations: [ DashboardHeroComponent, TestHostComponent ]
})
// create TestHostComponent instead of DashboardHeroComponent
fixture  = TestBed.createComponent(TestHostComponent);
testHost = fixture.componentInstance;
heroEl   = fixture.nativeElement.querySelector('.hero');
fixture.detectChanges(); // trigger initial data binding
```

Cette configuration du module de test montre trois différences importantes:

* Il déclare à la fois le `DashboardHeroComponent` et le `TestHostComponent`.
* Il crée le `TestHostComponent` au lieu du `DashboardHeroComponent`.
* Le `TestHostComponent` définit le `DashboardHeroComponent.hero` avec un binding.

`CreateComponent` renvoie une `fixture` contenant une instance de `TestHostComponent` au lieu d'une instance de `DashboardHeroComponent`.

La création de `TestHostComponent` a pour effet secondaire de créer un `DashboardHeroComponent`, car ce dernier apparaît dans le modèle du premier. La requête relative à l'élément hero (heroEl) le trouve toujours dans le DOM de test, mais plus en profondeur dans l'arborescence des éléments.

Les tests eux-mêmes sont presque identiques à la version autonome:

```js
it('should display hero name', () => {
  const expectedPipedName = testHost.hero.name.toUpperCase();
  expect(heroEl.textContent).toContain(expectedPipedName);
});

it('should raise selected event when clicked', () => {
  click(heroEl);
  // selected hero should be the same data bound hero
  expect(testHost.selectedHero).toBe(testHost.hero);
});
```

Seul le test d'événement sélectionné diffère. Cela confirme que le héros sélectionné `DashboardHeroComponent` trouve vraiment son chemin dans la liaison d'événement au component hôte.



### Routing Component

Un routing component est un component qui indique au routeur de naviguer vers un autre component. Le component de `DashboardComponent` est un routing compononent, car l'utilisateur peut accéder à `HeroDetailComponent` en cliquant sur l'un des HeroButtons du `Dashboard`.

Le routage est assez compliqué. Tester le  `DashboardComponent` peut sembler décourageant en partie parce qu’il implique le routeur, qu’il injecte avec `HeroService`.

```js
constructor(
  private router: Router,
  private heroService: HeroService) {
}
```



```typescript
gotoDetail(hero: Hero) {
  let url = `/heroes/${hero.id}`;
  this.router.navigateByUrl(url);
}
```

```typescript
const routerSpy = jasmine.createSpyObj('Router', ['navigateByUrl']);
const heroServiceSpy = jasmine.createSpyObj('HeroService', ['getHeroes']);

TestBed.configureTestingModule({
  providers: [
    { provide: HeroService, useValue: heroServiceSpy },
    { provide: Router,      useValue: routerSpy }
  ]
})
```

```js
it('should tell ROUTER to navigate when hero clicked', () => {

  heroClick(); // trigger click on first inner <div class="hero">

  // args passed to router.navigateByUrl() spy
  const spy = router.navigateByUrl as jasmine.Spy;
  const navArgs = spy.calls.first().args[0];

  // expecting to navigate to id of the component's first hero
  const id = comp.heroes[0].id;
  expect(navArgs).toBe('/heroes/' + id,
    'should nav to HeroDetail for first hero');
});
```



## Attribute Directive Testing

Une _directive d'attribut_ modifie le comportement d'un élément, d'un component ou d'une autre directive. Son nom reflète la manière dont la directive est appliquée: en tant qu'attribut sur un élément hôte.

`HighlightDirective`  définit la couleur d'arrière-plan d'un élément en fonction d'une couleur liée aux données ou d'une couleur par défaut (lightgray). Il définit également une propriété personnalisée de l'élément (customProperty) sur true pour aucune autre raison que celle de montrer qu'il le peut.

```js
import { Directive, ElementRef, Input, OnChanges } from '@angular/core';

@Directive({ selector: '[highlight]' })
/** Set backgroundColor for the attached element to highlight color
 *  and set the element's customProperty to true */
export class HighlightDirective implements OnChanges {

  defaultColor =  'rgb(211, 211, 211)'; // lightgray

  @Input('highlight') bgColor: string;

  constructor(private el: ElementRef) {
    el.nativeElement.style.customProperty = true;
  }

  ngOnChanges() {
    this.el.nativeElement.style.backgroundColor = this.bgColor || this.defaultColor;
  }
}
```

Il est utilisé dans toute l'application, peut-être plus simplement dans le component About:

```js
import { Component } from '@angular/core';
@Component({
  template: `
  <h2 highlight="skyblue">About</h2>
  <h3>Quote of the day:</h3>
  <twain-quote></twain-quote>
  `
})
export class AboutComponent { }
```

Tester l'utilisation spécifique de `HighlightDirective` dans `AboutComponent` ne nécessite que les techniques explorées ci-dessus (en particulier l'approche "test superficiel").

```js
beforeEach(() => {
  fixture = TestBed.configureTestingModule({
    declarations: [ AboutComponent, HighlightDirective],
    schemas:      [ NO_ERRORS_SCHEMA ]
  })
  .createComponent(AboutComponent);
  fixture.detectChanges(); // initial binding
});

it('should have skyblue <h2>', () => {
  const h2: HTMLElement = fixture.nativeElement.querySelector('h2');
  const bgColor = h2.style.backgroundColor;
  expect(bgColor).toBe('skyblue');
});
```

>  Toutefois, il est peu probable que le test d'un cas d'utilisation unique explore toute la gamme des capacités d'une directive. Trouver et tester tous les composants qui utilisent la directive est fastidieux, fragile et il est presque aussi improbable qu’il offre une couverture complète.
>
> Les tests de classe uniquement peuvent être utiles, mais les directives d'attribut comme celle-ci ont tendance à manipuler le DOM. Les tests unitaires isolés ne touchent pas le DOM et n'inspirent donc pas confiance dans l'efficacité de la directive.
>
> Une meilleure solution consiste à créer un composant de test artificiel qui montre toutes les manières d'appliquer la directive.

```js
@Component({
  template: `
  <h2 highlight="yellow">Something Yellow</h2>
  <h2 highlight>The Default (Gray)</h2>
  <h2>No Highlight</h2>
  <input #box [highlight]="box.value" value="cyan"/>`
})
class TestComponent { }
```

Voici quelques tests: 

```js
beforeEach(() => {
  fixture = TestBed.configureTestingModule({
    declarations: [ HighlightDirective, TestComponent ]
  })
  .createComponent(TestComponent);

  fixture.detectChanges(); // initial binding

  // all elements with an attached HighlightDirective
  des = fixture.debugElement.queryAll(By.directive(HighlightDirective));

  // the h2 without the HighlightDirective
  bareH2 = fixture.debugElement.query(By.css('h2:not([highlight])'));
});

// color tests
it('should have three highlighted elements', () => {
  expect(des.length).toBe(3);
});

it('should color 1st <h2> background "yellow"', () => {
  const bgColor = des[0].nativeElement.style.backgroundColor;
  expect(bgColor).toBe('yellow');
});

it('should color 2nd <h2> background w/ default color', () => {
  const dir = des[1].injector.get(HighlightDirective) as HighlightDirective;
  const bgColor = des[1].nativeElement.style.backgroundColor;
  expect(bgColor).toBe(dir.defaultColor);
});

it('should bind <input> background to value color', () => {
  // easier to work with nativeElement
  const input = des[2].nativeElement as HTMLInputElement;
  expect(input.style.backgroundColor).toBe('cyan', 'initial backgroundColor');

  // dispatch a DOM event so that Angular responds to the input value change.
  input.value = 'green';
  input.dispatchEvent(newEvent('input'));
  fixture.detectChanges();

  expect(input.style.backgroundColor).toBe('green', 'changed backgroundColor');
});


it('bare <h2> should not have a customProperty', () => {
  expect(bareH2.properties['customProperty']).toBeUndefined();
});
```

### Pipe Testing

Les _pipes_ sont faciles à tester sans les utilitaires de test Angular.

Une Pipe class possède une méthode, transform, qui manipule la valeur d'entrée en une valeur de sortie transformée. L'implémentation de transformation interagit rarement avec le DOM. La plupart des Pipe n’ont aucune dépendance d'Angular autre que les métadatas @Pipe et une interface.

Considérons un `TitleCasePipe` qui capitalise la première lettre de chaque mot. Voici une implémentation naïve avec une expression régulière.

```js
import { Pipe, PipeTransform } from '@angular/core';

@Pipe({name: 'titlecase', pure: true})
/** Transform to Title Case: uppercase the first letter of the words in a string.*/
export class TitleCasePipe implements PipeTransform {
  transform(input: string): string {
    return input.length === 0 ? '' :
      input.replace(/\w\S*/g, (txt => txt[0].toUpperCase() + txt.substr(1).toLowerCase() ));
  }
}
```

Tout ce qui utilise une expression régulière mérite d’être testé à fond. Utilisons Jasmine pour explorer les cas attendus et les cas extrêmes.

```js
describe('TitleCasePipe', () => {
  // This pipe is a pure, stateless function so no need for BeforeEach
  let pipe = new TitleCasePipe();

  it('transforms "abc" to "Abc"', () => {
    expect(pipe.transform('abc')).toBe('Abc');
  });

  it('transforms "abc def" to "Abc Def"', () => {
    expect(pipe.transform('abc def')).toBe('Abc Def');
  });

  // ... more tests ...
});
```

#### Write DOM tests too 

Ces tests ne peuvent pas dire si `TitleCasePipe` fonctionne correctement tel qu'appliqué dans les components de l'application.

Il faut penser à ajouter des tests de components tels que celui-ci:

```js
it('should convert hero name to Title Case', () => {
  // get the name's input and display elements from the DOM
  const hostElement = fixture.nativeElement;
  const nameInput: HTMLInputElement = hostElement.querySelector('input');
  const nameDisplay: HTMLElement = hostElement.querySelector('span');

  // simulate user entering a new name into the input box
  nameInput.value = 'quick BROWN  fOx';

  // dispatch a DOM event so that Angular learns of input value change.
  nameInput.dispatchEvent(newEvent('input'));

  // Tell Angular to update the display binding through the title pipe
  fixture.detectChanges();

  expect(nameDisplay.textContent).toBe('Quick Brown  Fox');
});
```

