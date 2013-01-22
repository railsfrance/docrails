Les méthodes de callback d'Active Record
=======================================
Ce guide va vous apprendre comment intervenir dans le cycle de vie de vos objets Active Record.

Après avoir lu ce guide, vous connaîtrez :
* Le cycle de vie des objets Active Record ;
* Comment créer des méthodes qui seront appelées lors d'évènements arrivant durant le cycle de vie d'un object Active Record ;
* Comment créer des classes spéciales qui encapsuleront des comportements génériques pour vos callbacks.

--------------------------------------------------------------------------------

Le Cycle de vie d'un Objet
---------------------------

Lors du fonctionnement normal d'une application Rails, des objets peuvent être crées, mis à jour et détruits. Active Record vous permet d'intervenir lors du <em>cycle de vie de l'objet</em> afin de pouvoir contrôler votre application et ses données.

Les méthodes de Callback vous permettent d'exécuter de la logique métier avant ou après l'alteration de l'état d'un objet.

Que sont les Méthodes de callback ?
-----------------------------------

Les méthodes de callback sont des méthodes qui sont appelées à certains moments dans le cycle de vie d'un objet. Avec les méthodes de callback, il est possible d'écrire du code qui sera executé lorsqu'un objet Active Record sera créé, sauvegardé, mis à jour, supprimé, validé ou chargé depuis la base de données.

### Enregistrement d'une Méthode de callback

Afin d'utiliser une méthode de callback, vous devez d'abord l'enregistrer. Vous pouvez enregistrer une méthode de callback soit de façon classique via une méthode d'instance, soit en utilisant une méthode de classe dite "macro".

```ruby
class User < ActiveRecord::Base
  validates :login, :email, presence: true

  before_validation :ensure_login_has_a_value

  protected
  def ensure_login_has_a_value
    if login.nil?
      self.login = email unless email.blank?
    end
  end
end
```

Les méthodes de classe dites "macro" peuvent aussi recevoir un bloc. Vous devriez uniquement utiliser cette méthode si le contenu de votre bloc ne dépasse pas une ligne.

```ruby
class User < ActiveRecord::Base
  validates :login, :email, presence: true

  before_create do |user|
    user.name = user.login.capitalize if user.name.blank?
  end
end
```

Les méthodes de callback peuvent aussi être enregistrées afin de ne s'exécuter qu'à certains moments dans le cycle de vie d'un objet.

```ruby
class User < ActiveRecord::Base
  before_validation :normalize_name, on: :create

  # :on Peut aussi recevoir un tableau.
  after_validation :set_location, on: [ :create, :update ]

  protected
  def normalize_name
    self.name = self.name.downcase.titleize
  end

  def set_location
    self.location = LocationService.query(self)
  end
end
```

Il est consideré comme une bonne pratique de déclarer les méthodes de callback comme privées ou protégées. Si ces méthodes sont laissées publiques, elles peuvent être appelées hors du modèle et violent ainsi le principe d'encapsulation objet.

Les Méthodes de Callback disponibles
------------------------------------

Voici une liste de toutes les Méthodes de callback Active Record disponibles, listées par leur ordre d'exécution :

### Création d'un Objet

* `before_validation`
* `after_validation`
* `before_save`
* `around_save`
* `before_create`
* `around_create`
* `after_create`
* `after_save`

### Mise à jour d'un Objet

* `before_validation`
* `after_validation`
* `before_save`
* `around_save`
* `before_update`
* `around_update`
* `after_update`
* `after_save`

### Destruction d'un Objet

* `before_destroy`
* `around_destroy`
* `after_destroy`

ATTENTION. `after_save` s'exécute à la création ***et*** à la mise à jour, mais toujours _après_ les méthodes de callback plus spécifiques `after_create` et `after_update`, peu importe l'ordre dans lequel les méthodes ont été déclarées.

### `after_initialize` et `after_find`

La méthode de callback `after_initialize` sera appelée à chaque instanciation d'un objet Active Record, soit en utilisant directement `new` ou quand un enregistrement est chargé depuis la base de données. Cela peut-être utile afin d'éviter d'outrepasser la méthode `initialize` d'un objet Active Record.

La méthode de callback `after_find` sera appelée à chaque fois qu'Active Record charge un enregistrement depuis la base de données. Elle sera appelée avant la méthode `after_initialize` si ces deux dernières sont definies.

Les méthodes `after_initialize` et `after_find` n'ont pas d'équivalents `before_*` mais peuvent être enregistrées comme n'importe quelle méthode de callback Active Record.

```ruby
class User < ActiveRecord::Base
  after_initialize do |user|
    puts "Vous avez initialisé un objet !"
  end

  after_find do |user|
    puts "Vous avez trouvé un objet !"
  end
end

>> User.new
Vous avez initialisé un objet !
=> #<User id: nil>

>> User.first
Vous avez trouvé un objet !
Vous avez initialisé un objet !
=> #<User id: 1>
```

Exécuter les méthodes de callback
---------------------------------

Les méthodes suivantes déclenchent l'exécution des méthodes de callback :

* `create`
* `create!`
* `decrement!`
* `destroy`
* `destroy!`
* `destroy_all`
* `increment!`
* `save`
* `save!`
* `save(validate: false)`
* `toggle!`
* `update`
* `update_attribute`
* `update_attributes`
* `update_attributes!`
* `valid?`

De plus, la méthode de callback `after_find` est exécutée par les méthodes suivantes :

* `all`
* `first`
* `find`
* `find_by_*`
* `find_by_*!`
* `find_by_sql`
* `last`

La méthode de callback `after_initialize` est exécutée à chaque fois qu'un nouvel objet de la classe est initialisé.

NOTE: Les méthodes `find_by_*` et `find_by_*!` sont des méthodes de recherche dynamiques génerées automatiquement pour chaque attributs de l'objet. Vous pouvez en apprendre plus sur ces méthodes de recherche dynamiques en cliquant [ICI](active_record_querying.html#dynamic-finders)

Skipping Callbacks
------------------

Just as with validations, it is also possible to skip callbacks. These methods should be used with caution, however, because important business rules and application logic may be kept in callbacks. Bypassing them without understanding the potential implications may lead to invalid data.

* `decrement`
* `decrement_counter`
* `delete`
* `delete_all`
* `increment`
* `increment_counter`
* `toggle`
* `touch`
* `update_column`
* `update_columns`
* `update_all`
* `update_counters`

Halting Execution
-----------------

As you start registering new callbacks for your models, they will be queued for execution. This queue will include all your model's validations, the registered callbacks, and the database operation to be executed.

The whole callback chain is wrapped in a transaction. If any _before_ callback method returns exactly `false` or raises an exception, the execution chain gets halted and a ROLLBACK is issued; _after_ callbacks can only accomplish that by raising an exception.

WARNING. Raising an arbitrary exception may break code that expects `save` and its friends not to fail like that. The `ActiveRecord::Rollback` exception is thought precisely to tell Active Record a rollback is going on. That one is internally captured but not reraised.

Relational Callbacks
--------------------

Callbacks work through model relationships, and can even be defined by them. Suppose an example where a user has many posts. A user's posts should be destroyed if the user is destroyed. Let's add an `after_destroy` callback to the `User` model by way of its relationship to the `Post` model:

```ruby
class User < ActiveRecord::Base
  has_many :posts, dependent: :destroy
end

class Post < ActiveRecord::Base
  after_destroy :log_destroy_action

  def log_destroy_action
    puts 'Post destroyed'
  end
end

>> user = User.first
=> #<User id: 1>
>> user.posts.create!
=> #<Post id: 1, user_id: 1>
>> user.destroy
Post destroyed
=> #<User id: 1>
```

Conditional Callbacks
---------------------

As with validations, we can also make the calling of a callback method conditional on the satisfaction of a given predicate. We can do this using the `:if` and `:unless` options, which can take a symbol, a string, a `Proc` or an `Array`. You may use the `:if` option when you want to specify under which conditions the callback **should** be called. If you want to specify the conditions under which the callback **should not** be called, then you may use the `:unless` option.

### Using `:if` and `:unless` with a `Symbol`

You can associate the `:if` and `:unless` options with a symbol corresponding to the name of a predicate method that will get called right before the callback. When using the `:if` option, the callback won't be executed if the predicate method returns false; when using the `:unless` option, the callback won't be executed if the predicate method returns true. This is the most common option. Using this form of registration it is also possible to register several different predicates that should be called to check if the callback should be executed.

```ruby
class Order < ActiveRecord::Base
  before_save :normalize_card_number, if: :paid_with_card?
end
```

### Using `:if` and `:unless` with a String

You can also use a string that will be evaluated using `eval` and hence needs to contain valid Ruby code. You should use this option only when the string represents a really short condition:

```ruby
class Order < ActiveRecord::Base
  before_save :normalize_card_number, if: "paid_with_card?"
end
```

### Using `:if` and `:unless` with a `Proc`

Finally, it is possible to associate `:if` and `:unless` with a `Proc` object. This option is best suited when writing short validation methods, usually one-liners:

```ruby
class Order < ActiveRecord::Base
  before_save :normalize_card_number,
    if: Proc.new { |order| order.paid_with_card? }
end
```

### Multiple Conditions for Callbacks

When writing conditional callbacks, it is possible to mix both `:if` and `:unless` in the same callback declaration:

```ruby
class Comment < ActiveRecord::Base
  after_create :send_email_to_author, if: :author_wants_emails?,
    unless: Proc.new { |comment| comment.post.ignore_comments? }
end
```

Callback Classes
----------------

Sometimes the callback methods that you'll write will be useful enough to be reused by other models. Active Record makes it possible to create classes that encapsulate the callback methods, so it becomes very easy to reuse them.

Here's an example where we create a class with an `after_destroy` callback for a `PictureFile` model:

```ruby
class PictureFileCallbacks
  def after_destroy(picture_file)
    if File.exists?(picture_file.filepath)
      File.delete(picture_file.filepath)
    end
  end
end
```

When declared inside a class, as above, the callback methods will receive the model object as a parameter. We can now use the callback class in the model:

```ruby
class PictureFile < ActiveRecord::Base
  after_destroy PictureFileCallbacks.new
end
```

Note that we needed to instantiate a new `PictureFileCallbacks` object, since we declared our callback as an instance method. This is particularly useful if the callbacks make use of the state of the instantiated object. Often, however, it will make more sense to declare the callbacks as class methods:

```ruby
class PictureFileCallbacks
  def self.after_destroy(picture_file)
    if File.exists?(picture_file.filepath)
      File.delete(picture_file.filepath)
    end
  end
end
```

If the callback method is declared this way, it won't be necessary to instantiate a `PictureFileCallbacks` object.

```ruby
class PictureFile < ActiveRecord::Base
  after_destroy PictureFileCallbacks
end
```

You can declare as many callbacks as you want inside your callback classes.

Transaction Callbacks
---------------------

There are two additional callbacks that are triggered by the completion of a database transaction: `after_commit` and `after_rollback`. These callbacks are very similar to the `after_save` callback except that they don't execute until after database changes have either been committed or rolled back. They are most useful when your active record models need to interact with external systems which are not part of the database transaction.

Consider, for example, the previous example where the `PictureFile` model needs to delete a file after the corresponding record is destroyed. If anything raises an exception after the `after_destroy` callback is called and the transaction rolls back, the file will have been deleted and the model will be left in an inconsistent state. For example, suppose that `picture_file_2` in the code below is not valid and the `save!` method raises an error.

```ruby
PictureFile.transaction do
  picture_file_1.destroy
  picture_file_2.save!
end
```

By using the `after_commit` callback we can account for this case.

```ruby
class PictureFile < ActiveRecord::Base
  attr_accessor :delete_file

  after_destroy do |picture_file|
    picture_file.delete_file = picture_file.filepath
  end

  after_commit do |picture_file|
    if picture_file.delete_file && File.exist?(picture_file.delete_file)
      File.delete(picture_file.delete_file)
      picture_file.delete_file = nil
    end
  end
end
```

The `after_commit` and `after_rollback` callbacks are guaranteed to be called for all models created, updated, or destroyed within a transaction block. If any exceptions are raised within one of these callbacks, they will be ignored so that they don't interfere with the other callbacks. As such, if your callback code could raise an exception, you'll need to rescue it and handle it appropriately within the callback.
