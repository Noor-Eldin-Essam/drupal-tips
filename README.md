# Drupal-Tips
Drupal CMS development tips

# Custom Module Developmet
- Module file structure
```
|-- modules
  | -- custom
    |-- hello_world
      |-- hello_world.info.yml
      |-- hello_world.routing.yml
      |-- src
        |-- Controller
```
 * .info.yml: 
 > Containes all information about module [name, description, core, version, type]
 ```yml
name: Hello World Module
description: Creates a page showing "Hello World".
package: Custom

type: module
core: 8.x
core_version_requirement: ^8 || ^9
 ```
* .routing.yml
> Containes custom routing refear to controller callback function
```yml
hello_world.my_page:
  path: '/mypage/page'
  defaults:
    _controller: '\Drupal\hello_world\Controller\HelloWorldController::myPage'
    _title: 'My first page in Drupal'
  requirements:
    _permission: 'access content'
```
<strong>hello_world.my_page</strong><br>
<p>
  This is the machine name of the route. By convention, route machine names should be module_name.sub_name. When other parts of the code need to refer to the route, they will use the machine name.
</p>
<strong>path</strong><br>
<p>This gives the path to the page on your site. Note the leading slash (/).</p>
<strong>defaults</strong><br>
<p>This describes the page and title callbacks. @todo: Where can these defaults be overridden?</p>
<strong>requirements</strong><br>
<p>This specifies the conditions under which the page will be displayed. You can specify permissions, modules that must be enabled, and other conditions.</p>

--------------
[**URL generation**](https://api.drupal.org/api/drupal/core%21lib%21Drupal%21Core%21Url.php/class/Url/8.2.x).
> When adding a URL into a string, for instance in a text description, core recommends we use the `\Drupal\Core\Url` class. 
> This has a few handy methods:
- Internal URL based on a route - **`Url::fromRoute()`**, Example: `Url::fromRoute('acquia_connector.settings')`
- Internal URL based on a path - **`Url::fromInternalUri()`**, Example: `Url::fromInternalUri('node/add')`
- External URL - **`Url::fromUri`**, Example: `Url::fromUri('https://www.acquia.com')`
> When you need to constrict and display the link as text you can use the `toString()` method: `Url::fromRoute('acquia_connector.settings')->toString()`.

**Show Image Field In Twig**
```php
{{ content.field_my_image_field|render|striptags('<img>')|trim }}
```

**Drupal 8 Js, how to create a global Drupal Js function**
```javascript
/**
 * Attaches the single_datetime behavior.
 *
 * @type {Drupal~behavior}
 */
(function ($, Drupal, drupalSettings) {
  'use strict';
  Drupal.behaviors.leafletCurrentLocation = {
    attach: function (context,drupalSettings) {
    // Code....
    }
  }
})(jQuery, Drupal, drupalSettings);
```
**Drupal 8^ Current User Object, useful methods and what they return**
```php
get current user in drupal 8

$current_user = \Drupal::currentUser();

$uid = $current_user->id();
// It returns user id of current user.

$user_mail = $current_user->getEmail();
// It returns user email id.

$user_display_name = $current_user->getDisplayName();
// It returns user display name.

$user_account_name = $current_user->getAccountName()
// It returns user account name.

$user_roles = $current_user->getRoles();
// It returns array of current user has.

$current_user->isAnonymous();
// It returns true or false.

$current_user->isAuthenticated();
// It returns true or false.

$current_user->getLastAccessedTime();
// It returns timestamp of last logged in time for this user
```
[**Drupal 8^: How to inject a Service into a Class (Controller, Form, Plugin Block, etc) and why not into another service Class**](https://pixelthis.gr/content/drupal-8-how-inject-service-class-controller-form-plugin-block-etc-and-why-not-another)
# Inject a Service in Drupal 8

Is a good practice to inject a service whenever is possible.

## You can verify the service name by:

### Looking at the `Drupal` Static Service Container wrapper class.
Reading the code on the `Drupal` Class you can find the `httpClient` method:
```
  /**
   * Returns the default http client.
   *
   * @return \GuzzleHttp\Client
   *   A guzzle http client instance.
   */
  public static function httpClient() {
    return static::getContainer()->get('http_client');
  }
```
> Drupal class => https://api.drupal.org/api/drupal/core%21lib%21Drupal.php/class/Drupal/8

### Taking advanage of Drupal Console debugging capabilities 
```
$ drupal container:debug 
```
If you do not want to see the full list of services you can use `| grep http`
```
$ drupal container:debug | grep http
```
But you may do not know the service name, then I higly recommend you to use peco interactive filtering tool
```
$ drupal container:debug | peco | awk -F ' ' '{print $1}' | xargs drupal container:debug
```
You can find peco at https://github.com/peco/peco

## Inject the service
Now that you know the service name `http_client` 

### Inject into a Class (Controller, Form, Plugin Block, etc)
```
 /**
   * Guzzle Http Client.
   *
   * @var GuzzleHttp\Client
   */
  protected $httpClient;

 /**
   * Constructs a new Class.
   *
   * @param \GuzzleHttp\Client $http_client
   *   The http_client.
   */
  public function __construct(
    Client $http_client
  ) {
    $this->httpClient = $http_client;
  }

  /**
   * {@inheritdoc}
   */
  public static function create(ContainerInterface $container) {
    return new static(
      $container->get('http_client')
    );
  }
```
### Inject a service into a service Class
```
// modules/custom/example/example.services.yml
services:
  example.default:
    class: Drupal\example\DefaultService
    arguments: ["@http_client"]

// modules/custom/example/src/DefaultService.php
 /**
   * GuzzleHttp\Client definition.
   *
   * @var GuzzleHttp\Client
   */
  protected $http_client;
  /**
   * Constructor.
   */
  public function __construct(Client $http_client) {
    $this->http_client = $http_client;
  }
```
**Drop custom table from database**.
```php
function hook_uninstall(){
    $table = 'your_table_name';
    $schema = Database::getConnection()->schema();
    $schema->dropTable($table);
}
```
**Get decrypted field values during entity insert/update hook**
> The following explains the situation on a `foo` node when attempting to get the value of the text field  
> `field_bar` during a form-alter, a node pre-save, a node insert, and a node update:
```php
/**
 * Implements hook_form_BASE_FORM_ID_alter().
 *
 * Alter the edit Foo node form.
 */
function mymodule_form_node_foo_edit_form_alter(&$form, FormStateInterface $form_state, $form_id) {
  /** @var \Drupal\node\Entity\Node $foo_node */
  $foo_node = $form_state->getFormObject()->getEntity();
  // Value 1 is correct.
  $bar_value_1 = $foo_node->field_bar->getString();
  // Value 2 is correct.
  $bar_value_2 = $foo_node->field_bar->getValue();
  // Value 3 is correct.
  $bar_value_3 = $foo_node->field_bar[0]->value;
  // Value 4 is correct.
  $bar_value_4 = $foo_node->get('field_bar')->getValue();
}

/**
 * Implements hook_ENTITY_TYPE_presave().
 */
function mymodule_node_presave(EntityInterface $entity) {
  // Value 1 is correct.
  $bar_value_1 = $entity->field_bar->getString();
  // Value 2 is correct.
  $bar_value_2 = $entity->field_bar->getValue();
  // Value 3 is correct.
  $bar_value_3 = $entity->field_bar[0]->value;
  // Value 4 is correct.
  $bar_value_4 = $entity->get('field_bar')->getValue();
}

/**
 * Implements hook_ENTITY_TYPE_insert().
 */
function mymodule_node_insert(EntityInterface $entity) {
  // Value 1 returns `[ENCRYPTED]`.
  $bar_value_1 = $entity->field_bar->getString();
  // Value 2 returns `[ENCRYPTED]`.
  $bar_value_2 = $entity->field_bar->getValue();
  // Value 3 returns `[ENCRYPTED]`.
  $bar_value_3 = $entity->field_bar[0]->value;
  // Value 4 returns `[ENCRYPTED]`.
  $bar_value_4 = $entity->get('field_bar')->getValue();

  /* @var $field_encrypt_process_entities \Drupal\field_encrypt\FieldEncryptProcessEntities */
  $field_encrypt_process_entities = \Drupal::service('field_encrypt.process_entities');
  $field_encrypt_process_entities->decryptEntity($entity);

  // Value is STILL `[ENCRYPTED]`.
  $bar_value_1 = $entity->field_bar->getString();
  // Value is STILL `[ENCRYPTED]`.
  $bar_value_2 = $entity->field_bar->getValue();
  // Value is STILL `[ENCRYPTED]`.
  $bar_value_3 = $entity->field_bar[0]->value;
  // Value is STILL `[ENCRYPTED]`.
  $bar_value_4 = $entity->get('field_bar')->getValue();
}

/**
 * Implements hook_ENTITY_TYPE_update().
 */
function mymodule_node_update(EntityInterface $entity) {
  // Value 1 returns `[ENCRYPTED]`.
  $bar_value_1 = $entity->field_bar->getString();
  // Value 2 returns `[ENCRYPTED]`.
  $bar_value_2 = $entity->field_bar->getValue();
  // Value 3 returns `[ENCRYPTED]`.
  $bar_value_3 = $entity->field_bar[0]->value;
  // Value 4 returns `[ENCRYPTED]`.
  $bar_value_4 = $entity->get('field_bar')->getValue();

  /* @var $field_encrypt_process_entities \Drupal\field_encrypt\FieldEncryptProcessEntities */
  $field_encrypt_process_entities = \Drupal::service('field_encrypt.process_entities');
  $field_encrypt_process_entities->decryptEntity($entity);

  // Value is STILL `[ENCRYPTED]`.
  $bar_value_1 = $entity->field_bar->getString();
  // Value is STILL `[ENCRYPTED]`.
  $bar_value_2 = $entity->field_bar->getValue();
  // Value is STILL `[ENCRYPTED]`.
  $bar_value_3 = $entity->field_bar[0]->value;
  // Value is STILL `[ENCRYPTED]`.
  $bar_value_4 = $entity->get('field_bar')->getValue();
}
```
**Check if current user is admin**.
```php
$user = \Drupal::currentUser()->getRoles();
  if(!in_array("administrator", $user) && $form_id == 'user_form') {
  	unset($form['account']);
  }
```
**Drupal 8 programmatically block and unblock a user**
```php
/** @var \Drupal\user\UserInterface $user */
$user = \Drupal\user\Entity\User::load(USER_ID_HERE);
$user->block();
// $user->activate();
$user->save();
```

**Code snippet that can be used to Log in user programmatically in Drupal 8**
```php
use Drupal\user\Entity\User;
use Symfony\Component\HttpFoundation\RedirectResponse;


$user = User::create([
  'name' => $userEmail,
  'mail' => $userEmail,
  'pass' => $password,
  'status' => 1,
]);
user_login_finalize($user);
$user_destination = \Drupal::destination()->get();
$response = new RedirectResponse($user_destination);
$response->send();
```

**Disable block caching in drupal 8**
```php
class MYCUSTOMBLOCK extends BlockBase {
  /**
   * {@inheritdoc}
   */
    public function build() {
        return array(
            '#markup' => ""
        );
    }
    /**
     * {@inheritdoc}
     */
    public function getCacheMaxAge() {
        return 0;
    }
}
```
**FORM**
```php
public function getFormId() { 
  return 'multistep_form_one'; 
}

public function bulidForm(array $form, FormStateInterface $form_state) { 
  $form['name'] = array( '#type' => 'textfield', 'title'=> 'name', ); 
  return $form; 
}

public function validateForm(array $form, FormStateInterface $form_state) { 
  if(!is_numeric($form_state->getValue('name')){
    $form_state->setErrorByName("Enter only alphabets");
  } 
}

public function submitForm(array &$form, FormStateInterface $form_state) {
  foreach ($form_state->getValues() as $key => $value) { 
    drupal_set_message($key . ': ' . $value);
  } 
}
```
**Paragraph Module**
> load the paragraphs content.
```php
$node  = \Drupal\node\Entity\Node::load(1);
$paragraph = $node->field_paragraph->getValue();
// Loop through the result set.
foreach ( $paragraph as $element ) {
  $p = \Drupal\paragraphs\Entity\Paragraph::load( $element['target_id'] );
  $text = $p->field_name->getValue();
}
```
**Profile Module**
```php
// Load profile by user id and profile type.
 $profile = \Drupal::entityTypeManager()
        ->getStorage('profile')
        ->loadByProperties([
          'uid' => $doctor_id,
          'type' => 'doctor_profile',
        ]);
 // Get filed data.
 $hcp_profile = reset($hcp_profiles);
 $address = $hcp_profile->get('field_user_address')->getValue();
 //Or
 $full_name = $profile->field_full_name->value;
```
**Set datetime-local html default value**
```html
<div class="form-group">
<label for="">موعد الاختبار</label>
<?php $datetime = new DateTime($row['start_date']); ?>
<input type="datetime-local" class="form-control" name="start_date"
  value="<?= $datetime->format('Y-m-d\TH:i:s'); ?>" required>
</div>
```

**Create node programmatically**
```php
Try this Code in MigrateContentController:

<?php

namespace Drupal\migrate_contents\Controller;
use Drupal\taxonomy\Entity\Term;
use Drupal\Core\Controller\ControllerBase;
use Drupal\node\Entity\Node;

class MigrateContentController extends ControllerBase {

 function migrateContent() {
 $updated = 0;
 $data = array(
   'type' => 'ip_range', 
   'title' => '192.168.7.100/24', 
   'field_ip_range_gw' => '192.168.7.100', 
   'field_ip_range_hidden' => '', 
   'field_ip_range_blocked' => '192.168.7.200', 
   'field_ip_range_access_type' => 'Blocked', 
   'field_ip_threshold' => '20', 
   'field_ip_range_sec_zone' => 'C2', 
   'field_ip_range_vlan_name' => 'Network58', 
   'field_ip_range_vlan_tag' => '4054', 
   'field_ip_range_comment' => 'Ip ranges from 192.168.7.100 to 192.168.7.255', 
 );
 $node = Node::create($data);
 $node->save();
 $updated++;

 return array(
   '#markup' => $updated,
 );
 }
}
```
## DRUPAL CONSOLE
**Installing**.
> composer require drupal/console:~1.0 --prefer-dist --optimize-autoloader --sort-packages

** Reset user password using uid**
```php
public function restPass()
  {
    $user = \Drupal\user\Entity\User::load(85);
    kint($user);
    die();
    $user->setPassword('admin');
    $user->save();
    return [
      '#type' => 'markup',
      '#markup' => 'Done'
    ];
  }
```

**Load node by entityQuery**
```php
$query = \Drupal::entityQuery('node')
    ->condition('status', 1)
    ->condition('changed', REQUEST_TIME, '<')
    ->condition('title', 'cat', 'CONTAINS')
    ->condition('field_tags.entity.name', 'cats');
$nids = $query->execute();
```
** Redirect after login**
```php
use Drupal\Core\Form\FormStateInterface;
use Drupal\Core\Url;

/**
 * Implements hook_form_FORM_ID_alter().
 */
function MYCUSTOMMODULE_form_user_login_form_alter(&$form, FormStateInterface $form_state, $form_id) {
  $form['#submit'][] = 'MYCUSTOMMODULE_user_login_form_submit';
}

/**
 * Custom submit handler for the login form.
 */
function MYCUSTOMMODULE_user_login_form_submit($form, FormStateInterface $form_state) {
  $url = Url::fromRoute('/your/destination/path');
  $form_state->setRedirectUrl($url);
}
```
** Middlware for redirect issues**
```php
  $middleware = \Drupal::service('http_middleware.oncotogther_core');
 
// module.service.yml
services:
  http_middleware.oncotogther_core:
    class: Drupal\oncotogther_core\RedirectMiddleware
    tags:
      - { name: http_middleware}

<?php

// Service class.
namespace Drupal\oncotogther_core;

use Symfony\Component\HttpFoundation\Request;
use Symfony\Component\HttpFoundation\Response;
use Symfony\Component\HttpKernel\HttpKernelInterface;
use Symfony\Component\HttpFoundation\RedirectResponse;
use Drupal\Core\Routing\TrustedRedirectResponse;
use Drupal\Core\Url;

/**
 * Executes redirect before the main kernel takes over the request.
 */
class RedirectMiddleware implements HttpKernelInterface {

  /**
     * The wrapped HTTP kernel.
     *
     * @var \Symfony\Component\HttpKernel\HttpKernelInterface
     */
  protected $httpKernel;

  /**
     * The redirect URL.
     *
     * @var \Symfony\Component\HttpFoundation\RedirectResponse
     */
  protected $redirectResponse;

  /**
     * Constructs a RedirectMiddleware
     * object.
     *
     * @param \Symfony\Component\HttpKernel\HttpKernelInterface $http_kernel
     *   The decorated kernel.
     */
  public function __construct(HttpKernelInterface $http_kernel) {
        $this->httpKernel = $http_kernel;
      }

  /**
     * {@inheritdoc}
     */
  public function handle(Request $request, $type = self::MASTER_REQUEST, $catch = TRUE) {
        $response = $this->httpKernel->handle($request, $type, $catch);
        return $this->redirectResponse ?: $response;
  }

  /**
     * Stores the requested redirect response.
     *
     * @param \Symfony\Component\HttpFoundation\RedirectResponse|null $redirectResponse
     *   Redirect response.
     */
  public function setRedirectResponse(?RedirectResponse $redirectResponse) {
        $this->redirectResponse = $redirectResponse;
  }

}

// USING
$middleware = \Drupal::service('http_middleware.oncotogther_core');
$response = new RedirectResponse(Url::fromUserInput($url)->toString());
$middleware->setRedirectResponse($response);
```
