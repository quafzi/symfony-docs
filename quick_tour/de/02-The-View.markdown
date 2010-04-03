Eine schnelle Tour durch Symfony 2.0: Der View
==============================================


Decorating Templates
--------------------

Häufig enthalten die Templates eines Projektes globale Elemente, so zum Beispiel
Header und Footer. In Symfony wird dieses Problem anders gehandhabt: Templates
können durch andere Templates gefüllt werden.

Schauen wir uns doch einmal die Datei `layout.php`an:

    [php]
    # src/Application/HelloBundle/Resources/views/layout.php
    <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
    <html>
      <head>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
      </head>
      <body>
        <?php $view->slots->output('_content') ?>
      </body>
    </html>

Das `index`-Template wird durch die `layout.php` gefüllt. Das wird durch den
Aufruf von `extend()` ermöglicht:

    [php]
    # src/Application/HelloBundle/Resources/views/Hello/index.php
    <?php $view->extend('HelloBundle::layout') ?>

    Hello <?php echo $name ?>!

Die Notation `HelloBundle::layout` kommt uns durchaus bekannt vor, oder? Es ist
die gleiche Notation, die genutzt wird, um ein Template zu referenzieren. Der
Teil `::` bedeutet einfach, dass der Controller leer ist und die entsprechende
Datei direkt in `views/` liegt.

Der Ausdruck `$view->slots->output('_content')` wird durch den Inhalt des
Kindtemplates ersetzt, in diesem Fall durch die `index.php` (mehr dazu im
nächsten Abschnitt).

Wie man sieht, stellt Symfony eine Methode auf einem mysteriösen `$view`-Objekt
zur Verfügung. Im Template bezieht sich `$view` auf ein spezielles Objekt, das
ein Bündel von Methoden und Eigenschaften bereitstellt, die Template-Engine
ausmachen.

Symfony unterstützt mehrere Ebenen der Gestaltung: Ein Layout kann durch ein
anderes gestaltet werden. Diese Technik ist insbesondere für große Projekte
nützlich und entfaltet ihre volle Kraft, wenn sie in Kombination mit Slots
verwendet wird.

Slots
-----
Was ist ein Slot? Ein Slot ist ein Codeschnipsel, der in einem Template
definiert wird und für die Gestaltung von Templates wiederverwendet werden
kann. Wir werden nun im index-Template einen `title`-Slot definieren:

    [php]
    # src/Application/HelloBundle/Resources/views/Hello/index.php
    <?php $view->extend('HelloBundle::layout') ?>

    <?php $view->slots->set('title', 'Hello World app') ?>

    Hello <?php echo $name ?>!

Und nun verändern wir das Layout so, dass der Titel im Header ausgegeben wird:

    [php]
    # src/Application/HelloBundle/Resources/views/layout.php
    <html>
      <head>
        <title><?php $view->slots->output('title', 'Default Title') ?></title>
        <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
      </head>
      <body>
        <?php $view->slots->output('_content') ?>
      </body>
    </html>

Die `output()`-Methode fügt den Inhalt eines Slots ein und nimmt einen
Standardwert für den Fall entgegen, dass der Slot nicht definiert ist.
Der String `_content` bezeichnet einen speziellen Slot, der das gerenderte
Kindtemplate enthält.

Für große Slots gibt es auch eine erweiterte Syntax:

    [php]
    <?php $view->slots->start('title') ?>
      Some large amount of HTML
    <?php $view->slots->stop() ?>

Einbinden anderer Templates
---------------------------

Die einfachste Art, ein Codeschnipsel in mehreren verschiedenen Templates
gemeinsam zu verwenden, ist das Definieren eines Templates, das in andere
eingebunden werden kann.

Legen wir also ein `hello.php`-Template an:

    [php]
    # src/Application/HelloBundle/Resources/views/Hello/hello.php
    Hello <?php echo $name ?>!

Und nun binden wir dieses in das `index.php`-Template ein:

    [php]
    # src/Application/HelloBundle/Resources/views/Hello/index.php
    <?php $view->extend('HelloBundle::layout') ?>

    <?php echo $view->render('HelloBundle:Hello:hello', array('name' => $name)) ?>

Die `render()`-Methode wertet den Inhalt eines anderen Templates aus und gibt
ihn zurück (Das ist die gleiche Methode, die auch im Controller verwendet wird).

Einbinden anderer Actions
-------------------------

Was aber tun, wenn man das Ergebnis einer anderen Action in einem Template
ausgeben möchte? Dieser Fall tritt häufig ein, wenn man mit Ajax arbeitet
oder wenn im eingebetteten Template Variablen verwendet werden sollen, die
das Haupttemplate nicht enthält.

Hast du eine `fancy` Action und möchtest sie in das `index`-Template
einbinden, so verwendest du einfach den folgenden Code:

    [php]
    # src/Application/HelloBundle/Resources/views/Hello/index.php
    <?php $view->actions->output('HelloBundle:Hello:fancy', array('name' => $name, 'color' => 'green')) ?>

Hier bezieht sich die Zeichenkette `HelloBundle:Hello:fancy` auf die
`fancy`-Action des `Hello`-Controllers:

    [php]
    # src/Application/HelloBundle/Controller/HelloController.php
    class HelloController extends Controller
    {
      public function fancyAction($name, $color)
      {
        // create some object, based on the $color variable
        $object = ...;

        return $this->render('HelloBundle:Hello:fancy', array('name' => $name, 'object' => $object));
      }

      // ...
    }

Du solltest bedenken, dass diese Technik zwar sehr kraftvoll ist, andererseits
aber die Geschwindigkeit senken kann, da intern ein Unterrequest ausgeführt
wird. Daher sollten - wenn möglich - schnellere Alternativen eingesetzt werden.

Wo aber ist die Eigenschaft `$view->actions` definiert? Ähnlich wie bei
`$view->slots` handelt es sich um einen der Template-Helper, über die wir im
nächsten Abschnitt mehr erfahren werden.

Template-Helper
---------------

The Symfony templating system can be easily extended via helpers. Helpers are
PHP objects that provide features useful in a template context. `actions` and
`slots` are just two of the built-in Symfony helpers.

### Links between Pages

Speaking of web applications, creating links between different pages is a
must. Instead of hardcoding URLs in templates, the `router` helper knows how
to generate URLs based on the routing configuration. That way, all your URLs
can be easily updated by changing the configuration.

    [php]
    <a href="<?php echo $view->router->generate('hello', array('name' => 'Thomas')) ?>">
      Greet Thomas!
    </a>

The `generate()` method takes the route name and an array of values as
arguments. The route name is the main key under which routes are referenced
and the values should at least cover the route pattern placeholders:

    [yml]
    # src/Application/HelloBundle/Resources/config/routing.yml
    hello: # The route name
      pattern:  /hello/:name
      defaults: { _bundle: HelloBundle, _controller: Hello, _action: index }

### Using Assets: images, JavaScripts, and stylesheets

What would the Internet be without images, JavaScripts, and stylesheets?
Symfony provides three helpers to deal with them easily: `assets`,
`javascripts`, and `stylesheets`.

    [php]
    <link href="<?php echo $view->assets->getUrl('css/blog.css') ?>" rel="stylesheet" type="text/css" />

    <img src="<?php echo $view->assets->getUrl('images/logo.png') ?>" />

The `assets` helpers main purpose is to make your application more portable.
Thanks to it, you can move the application root directory anywhere under your
web root directory without changing anything in your templates code.

Similarly, you can manage your stylesheets and JavaScripts with the
`stylesheets` and `JavaScripts` helpers:

    [php]
    <?php $view->javascripts->add('js/product.js') ?>
    <?php $view->stylesheets->add('css/product.css') ?>

The `add()` method defines dependencies. To actually output these assets, you
need to also add the following code in your main layout:

    [php]
    <?php echo $view->javascripts ?>
    <?php echo $view->stylesheets ?>

Final Thoughts
--------------

The Symfony templating system is simple and powerful. Thanks to layouts,
slots, templating and action inclusions, it is very easy to organize your
templates in a logical and extensible way. In the later part, you will learn
how to configure the default behavior of the templating system and how to
extend it by adding new helpers.

You are only working with Symfony since about 20 minutes, and you can already
do pretty amazing stuff with it. That's the power of Symfony. Learning the
basics is easy, and you will soon learn that this simplicity is hidden under a
very flexible architecture.

But I get ahead of myself. First, you need to learn more about the controller
and that's exactly the topic of the next part of this tutorial. Ready for
another 10 minutes with Symfony?
