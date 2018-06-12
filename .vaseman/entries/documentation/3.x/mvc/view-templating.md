---
layout: documentation.twig
title: View and Templating

---

## Use View Object

In controller, you can do anything you want, but if you hope to render some template, the View object will help you. 
In this case, we create a default `HtmlView` to render template:

```php
// /src/Flower/Controller/Sakuras/GetController.php

class GetController extends Controller
{
	protected function doExecute()
	{
		$view = new HtmlView();

		return $view->render();
	}
}
```

Then create a php file in `/templates/default.php`. This template will be rendered.

```php
<?php
// templates/default.php
?>
Hello World
```

### Push Controller Information into View

But this is not a good position to locate template, we will hope it at `templates/flower/sakuras`.
So add this code to push controller config into View, View will know every thing about Controller:

```php
// /src/Flower/Controller/Sakuras/GetController.php

$view = new HtmlView();

// Push config into View
$view->setConfig($this->config);

return $view->render();
```

Move template file to `/templates/flower/sakuras/default.php`, and Windwalker will found it.

### Use Package Templates

If your View is in a package, you can put your template at `{Package}/Templates/{view}`, For example, we move `default.php` 
to `/src/Flower/Templates/sakuras`, View will find this position priority than root templates folder. 
So we can make our templates following package folders.

### The Ordering of Template Paths
 
Windwalker will follow this orders to find templates, you can override any template in a higher priority position:

```
[0] => {ROOT}/src/{package}/Templates/{view}/{locale}
[1] => {ROOT}/src/{package}/Templates/{view}
[2] => {ROOT}/src/{package}/Templates
[3] => {ROOT}/templates/{package}/{view}
[4] => {ROOT}/templates/{package}
[5] => {ROOT}/templates
[6] => {ROOT}/vendor/windwalker/core/src/Core/Resources/Templates
```

### Multilingual Paths

If you set locale in config `language.locale`, the current and default locale will auto added into paths with high priority:

```
[0] => /src/{package}/Templates/{view}/zh-TW  <--- current locale
[1] => /src/{package}/Templates/{view}/en-GB  <--- default locale
[2] => /src/{package}/Templates/{view}
[3] => /src/{package}/Templates
[4] => /templates/{package}/{view}
[5] => /templates/{package}
[6] => /templates
[7] => /vendor/windwalker/core/src/Core/Resources/Templates
```

### Use other layouts

The `foo.bar.baz` will matched `foo/bar/baz.php` file. If you didn't set any layout name, `default` will instead.

```php
// Will find: edit.php
$view->setLayout('edit')->render();

// Will find: foo/bar/baz.php
$view->setLayout('foo.bar.baz')->render();
```

### Custom Template Paths

Windwalker View use `SplPriorityQueue` to sort paths, if we want to add path, we should provide the priority flag.

```php
use Windwalker\Utilities\Queue\Priority;

// ...

$view->addPath('/my/template/path1', Priority::HIGH);
$view->addPath('/my/template/path2', Priority::NORMAL);
```

### Add Global Paths

Add paths to global that we don't need to set it every time, you must run this code before controller executed,
for example, you can run in `onAfterInitialise` event (See [Event](../more/event.html)) , `YourPackage::initialise()` or `Application::initialise()`:

```php
// Add global paths
\Windwalker\Core\Renderer\RendererHelper::addGlobalPath('/my/path', Priority::ABOVE_NORMAL);
```

See also: [Windwalker View](https://github.com/ventoviro/windwalker/tree/staging/src/View#htmlview)
/ [SplPriorityQueue](http://php.net/manual/en/class.splpriorityqueue.php)

### Add Data

View can maintain some data and use it in template:

```php
// Set it when construct
$view = new PhpHtmlView(array('foo' => 'bar'));

// Use setter
$view->set('foo', 'bar');

// Use Array Access
$view['foo'] = 'bar';
```

Then we can get this variables in template:

```php
<?php
// templates/flower/sakuras/default.php

?>
Hello <?php echo $foo;?>
```

## Create View Classes

We are still using default View, but it is not so customizable. Let's extend it to a new View object:

```php
<?php
// src/Flower/View/SakurasHtmlView.php

namespace Flower\View;

use Windwalker\Core\View\HtmlView;

class SakurasHtmlView extends HtmlView
{
	protected function prepareData($data)
	{
		$data->created = $data->created->format('Y-m-d');
	}
}
```

Always remember add `Html` in view name, sometimes we will need `JsonView` or `XmlView`.
Now we can create this view in controller:

```php
<?php
// /src/Flower/Controller/Sakuras/GetController.php

namespace Flower\Controller\Sakuras;

use Flower\View\SakurasHtmlView;
use Windwalker\Core\Controller\Controller;

class GetController extends Controller
{
	protected function doExecute()
	{
		$view = $this->getView('Sakuras');

		$view['created'] = new \DateTime('now');

		return $view->setLayout('sakuras')->render();
	}
}
```

The purpose of custom View object is that we can set data format in it, so controller dose not need to worry about 
how to show data with formatted, just consider how to send data into View and redirect pages.

## PHP Engine

PHP engine use [Windwalker Renderer](https://github.com/ventoviro/windwalker-renderer) to render page, 
this package provides a simple interface similar to Twig that support template extending.

### Include Sub Template

Use `load()` to load other template file as a block. The first argument is file path, the second argument is new data
to merge with original data.

```php
echo $this->load('sub.template', array('bar' => 'baz'));
```

Example to load `foo/article.php`:

```html
<?php
// foo/article.php
?>
<h1><?php echo $this->escape($title); ?></h1>

<?php foreach ($data->articles as $article): ?>
    <?php echo $this->load('foo.article', array('bar' => 'baz')); ?>
<?php endforeach; ?>
```

### Extends Parent Template

In Windwalker Renderer, there is a powerful function like Twig or Blade, we provide `extend()` method to extends
parent template. (`extends` in php is a reserved string, so we can only use `extend`)

For example, this is the parent `_global/html.php` template:

```html
<!-- _global/html.php -->
<Doctype html>
<html>
<head>
    <title><?php $this->block('title');?>Home<?php $this->endblock(); ?></title>
</head>
<body>
    <div class="container">
    <?php $this->block('body');?>
        <h2>Home page</h2>
    <?php $this->endblock(); ?>
    </div>
</body>
</html>
```

And we can extends it in our View:

```html
<?php
// foo/article.php
$this->extend('_global.html');
?>

<?php $this->block('title');?>Article<?php $this->endblock(); ?>

<?php $this->block('body');?>
    <article>
        <h2>Article</h2>
        <p>FOO</p>
    </article>
<?php $this->endblock(); ?>
```

The result will be:

```html
<Doctype html>
<html>
<head>
    <title>Article</title>
</head>
<body>
    <div class="container">
        <article>
            <h2>Article</h2>
            <p>FOO</p>
        </article>
    </div>
</body>
</html>
```

### Show Parent

We can echo parent data in a block:

```html
<?php $this->block('body');?>
    <?php echo $this->parent(); ?>
    <article>
        <h2>Article</h2>
        <p>FOO</p>
    </article>
<?php $this->endblock(); ?>
```

Result:

```html
<h2>Home page</h2>
<article>
    <h2>Article</h2>
    <p>FOO</p>
</article>
```

See: [Windwalker Renderer](https://github.com/ventoviro/windwalker-renderer#windwalker-renderer)

## Blade & Edge Engine

Blade is a simple, yet powerful templating engine provided with [Laravel](http://laravel.com/). It is driven by template inheritance and sections. 
All Blade templates should use the `.blade.php` extension.

There are 2 ways to use Blade engine in View, first is directly create it.

```php
use Windwalker\Core\View\BladeHtmlView;

// Use Blade view
$view = $this->getView('sakura', 'html', 'edge');

$view['foo'] = $bar;

echo $view->setLayout('flower.sakura')->render();
```

The second is extend it.

```php
use Windwalker\Core\View\HtmlView;

class SakuraHtmlView extends HtmlView
{
    protected $renderer = 'edge'; // Or RendererHelper::EDGE
}

$view = $this->getView('sakura', 'html');

echo $view->setLayout('flower.sakura')->render();
```

### Get Data in Edge Engine

In Blade template we don't need to use `$data`, all properties are at top level:

```php
{{ $item->title }}

{!! $uri['base.path'] !!}
```

See [Edge Templates](edge.html) [Blade Templates](hhttps://laravel.com/docs/5.2/blade)

## Twig Engine

Twig is a well-known template language for PHP, created by Sensio. It uses a syntax similar to the Django and Jinja template languages.

There are 2 ways to use Twig engine, similar to Blade engine:

```php
$view = $this->getView('sakura', 'html', 'twig');
```

OR

```php
use Windwalker\Core\View\HtmlView;

class SakuraHtmlView extends HtmlView
{
    protected $renderer = 'twig';
}

$view = $this->getView('sakura', 'html');
```

### How to Use Twig

See: [Twig Documentation](http://twig.sensiolabs.org/documentation)

## Global Variables

There is some global variables will auto set into View template, you can easily use them:

| Name | Description |
| ---- | ----------- |
| `view` | The view object self. |
| `helper` | the helper proxy object to help us quickly use other helper object. |
| `uri` | The uri registry object, see [Uri & Route Building](./uri-route-building.html) |
| `app` | The global application object. |
| `package` | The current package pbject. |
| `router` | The router object of current package. |
| `messages` | The flash messages. |
| `translator` | The translator object. |
| `datetime` | The Datetime object of current time. |
| `widget` | Instance of WidgetHelper to quickly render widget. |
| `asset` | AssetManager object. |

## Use Json, Xml or Other Format View

Sometimes we will hope a page can return multiple formats and control by HTTP query (`&format=xxx`).
We can create multiple Views with different formats to return data.

For example, we create a HTML class.

```php
// src/Flower/View/Sakura/SakuraHtmlView.php

namespace Flower\View\Sakura;

class SakurasHtmlView extends HtmlView
{
	/**
	 * @param \Windwalker\Data\Data $data
	 *
	 * @return  void
	 */
	protected function prepareData($data)
	{
		$data->items = $this->repository->getItems();
		$data->pagination = new Pagination($this->repository->get('page'));
		$data->title = 'Sakuras List';
		$data->metadata = 'Sakuras metadata';
	}
} 
```

And a Json view class.

```php
namespace Asuka\Flower\View\Sakuras;

use Windwalker\Core\View\StructureView;

class SakurasJsonView extends StructureView
{
	/**
	 * @param \Windwalker\Structure\Structure $structure
	 *
	 * @return  void
	 */
	protected function prepareData($structure)
	{
		$page = $this->repository->get('page');

		$structure->set('items', $this->repository->getItems());
		$structure->set('next', $this->router->to('sakuras')->page($page));
	}
}
```

You can notice that Html view and Json view prepared different data for their usage, then we must load them in controller:

```php
class GetController extends AbstractController
{
    public function doExecute()
    {
        $format = $this->input->get('format', 'html');
        
        $view = $this->getView('Sakuras', $format);

        switch ($format)
        {
            case 'html':
                $this->response = new HtmlResponse();
                break;

            case 'json':
                $this->response = new JsonResponse();
                break;
                
            // More formats...
        }

        // Just return view object, application will auto convert it to string.
        return $view;
    }
}
```

In the example, a page will use different format by URL `format=html|json` and the destination view returned different content for client.
 
You can create your `ResponseFactory::create($format)` and every thing will be automatically.

## Helpers

### HTML Escape

Use `echo html_escape($string)` to escape HTML in PHP. If you want to convert `\n` to `<br>`, add second argument as TRUE.

```php
<?php echo html_escape($string, true); ?>
```

### Edge HTML Attributes Binding

Use `@attr()` to bind an attribute to variables:

```html
<input type="text" @attr('required', $isRequired) />
```

The binding rule please see: https://github.com/ventoviro/windwalker-dom#attributes
