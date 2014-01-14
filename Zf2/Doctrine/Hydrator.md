##Hydrator

Traduzione del seguente articolo [https://github.com/doctrine/DoctrineModule/edit/master/docs/hydrator.md](https://github.com/doctrine/DoctrineModule/blob/master/docs/hydrator.md)

Gli hydrators sono semplici oggetti che permettono di convertire una matrice di dati in un oggetto 
("hydrating") e di riconvertire un oggetto in un array ("extracting").

####Creare un hydrator

Per creare un Doctrine Hydrator abbimo bisogno soltanto di una cosa: un object manager (chiamato anche Entity Manager in Doctrine ORM)

```php
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($objectManager);
```

####Esempio 1: semplice entità senza associazioni

```php
namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
class City
{
    /**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    protected $id;

    /**
     * @ORM\Column(type="string", length=48)
     */
    protected $name;

    public function getId()
    {
        return $this->id;
    }

    public function setName($name)
    {
        $this->name = $name;
    }

    public function getName()
    {
        return $this->name;
    }
}
```

Ora andiamo ad usare il Doctrine hydrator

```php
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$city = new City();
$data = array(
    'name' => 'Paris'
);

$city = $hydrator->hydrate($data, $city);

echo $city->getName(); // prints "Paris"

$dataArray = $hydrator->extract($city);
echo $dataArray['name']; // prints "Paris"
```

####Example 2 : associazioni OneToOne/ManyToOne

DoctrineModule hydrator è particolarmente utile quando si tratta di associazioni (OnetoOne, OneToMany, ManyToOne) e
si integra perfettamente con la logica Form/Fieldset ([vedi](http://framework.zend.com/manual/2.0/en/modules/zend.form.collections.html)).

Facciamo un semplice esempio con un BlogPost e un'entità User per illustrare associazione OneToOne:

```php

namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
class User
{
    /**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    protected $id;

    /**
     * @ORM\Column(type="string", length=48)
     */
    protected $username;

    /**
     * @ORM\Column(type="string")
     */
    protected $password;

    public function getId()
    {
	return $this->id;
    }

    public function setUsername($username)
    {
    	$this->username = $username;
    }

    public function getUsername()
    {
    	return $this->username;
    }

    public function setPassword($password)
    {
    	$this->password = $password;
    }

    public function getPassword()
    {
    	return $this->password;
    }
}
```

E l'entità BlogPost, con una associazione ManyToOne:

```php

namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
class BlogPost
{
    /**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    protected $id;

    /**
     * @ORM\ManyToOne(targetEntity="Application\Entity\User")
     */
    protected $user;

    /**
     * @ORM\Column(type="string")
     */
    protected $title;

    public function getId()
    {
        return $this->id;
    }

    public function setUser(User $user)
    {
    	$this->user = $user;
    }

    public function getUser()
    {
    	return $this->user;
    }

    public function setTitle($title)
    {
    	$this->title = $title;
    }

    public function getTitle()
    {
    	return $this->title;
    }
}
```

Ci sono due casi di utilizzo che possono verificarsi durante l'utilizzo di un'associazione OneToOne: l'entità toOne (nel nostro caso, l'utente) può esistere di già, 
oppure può anche essere creata. Il DoctrineHydrator supporta nativamente entrambi i casi.

#####Entità esistente nell'associazione

Quando l'entità dell'associazione esiste già, quello che dobbiamo fare è semplicemente dare l'identificativo dell'associazione:

```php
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$blogPost = new BlogPost();
$data = array(
	'title' => 'The best blog post in the world !',
	'user'  => array(
		'id' => 2 // Written by user 2
	)
);

$blogPost = $hydrator->hydrate($data, $blogPost);

echo $blogPost->getTitle(); // prints "The best blog post in the world !"
echo $blogPost->getUser()->getId(); // prints 2
```

**NOTE** : quando si utilizza l'associazione con chiave primaria non composta, è possibile riscrivere più succintamente:

```php
$data = array(
	'title' => 'The best blog post in the world !',
	'user'  => 2
);
```

##### Entità non esistente nell'associazione

Se l'entità dell'associazione non esiste, abbiamo solo bisogno di dare l'oggetto dato:

```php
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$blogPost = new BlogPost();
$user = new User();
$user->setUsername('bakura');
$user->setPassword('p@$$w0rd');

$data = array(
	'title' => 'The best blog post in the world !',
	'user'  => $user
);

$blogPost = $hydrator->hydrate($data, $blogPost);

echo $blogPost->getTitle(); // prints "The best blog post in the world !"
echo $blogPost->getUser()->getId(); // prints 2
```

Perchè funzioni, è necessario anche modificare leggermente la mappatura, in modo che Doctrine possa persistere le nuove entità sulle
associazioni (notare l'opzione cascade sull'associazione OneToMany):

```php

namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
class BlogPost
{
    /** .. */

    /**
     * @ORM\ManyToOne(targetEntity="Application\Entity\User", cascade={"persist"})
     */
    protected $user;

    /** … */
}
```

E' anche possibile utilizzare un fieldset nidificato per i dati utente. L'hydrator
utilizzerà i dati di mappatura per determinare gli identificatori per il rapporto toOne e
tentarà di trovare il record esistente o instanzierà una nuova istanza di destinazione che sarà
idratata prima che sia passata all'entità BlogPost.

```php
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager, 'Application\Entity\BlogPost');
$blogPost = new BlogPost();

$data = array(
    'title' => 'Art thou mad?',
    'user' => array(
        'id' => '',
        'username' => 'willshakes',
        'password' => '2BorN0t2B'
    )
);

$blogPost = $hydrator->hydrate($data, $blogPost);

echo $blogPost->getUser()->getUsername(); // prints willshakes
echo $blogPost->getUser()->getPassword(); // prints 2BorN0t2B
```

####Esempio 3: associazioni OneToMany

Il DoctrineModule hydrator gestisce anche le relazioni OneToMany (quando usa l'elemento `Zend\Form\Element\Collection`). Fare riferimento alla [documentazione di Zend Framework 2] (http://framework.zend.com/manual/2.0/en/modules/zend.form.collections.html) per saperne di più sulla Collection.

Vediamo un semplice esempio: un BlogPost e delle entità Tag

```php

namespace Application\Entity;

use Doctrine\Common\Collections\ArrayCollection;
use Doctrine\Common\Collections\Collection;
use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
class BlogPost
{
    /**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    protected $id;

    /**
     * @ORM\OneToMany(targetEntity="Application\Entity\Tag", mappedBy="blogPost")
     */
    protected $tags;

    /**
     * Never forget to initialize all your collections !
     */
	public function __construct()
	{
		$this->tags = new ArrayCollection();
	}

    public function getId()
    {
   		return $this->id;
    }

	public function addTags(Collection $tags)
	{
		foreach ($tags as $tag) {
			$tag->setBlogPost($this);
			$this->tags->add($tag);
		}
	}

	public function removeTags(Collection $tags)
	{
		foreach ($tags as $tag) {
			$tag->setBlogPost(null);
			$this->tags->removeElement($tag);
		}
	}

    public function getTags()
    {
    	return $this->tags;
    }
}
```

E l'entità Tag:

```php

namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
class Tag
{
	/**
     * @ORM\Id
     * @ORM\Column(type="integer")
     * @ORM\GeneratedValue(strategy="AUTO")
     */
    protected $id;

    /**
     * @ORM\ManyToOne(targetEntity="Application\Entity\BlogPost", inversedBy="tags")
     */
	protected $blogPost;

	/**
	 * @ORM\Column(type="string")
	 */
	protected $name;

    public function getId()
    {
   		return $this->id;
    }

	/**
	 * Allow null to remove association
	 */
	public function setBlogPost(BlogPost $blogPost = null)
	{
		$this->blogPost = $blogPost;
	}

    public function getBlogPost()
    {
    	return $this->blogPost;
    }

    public function setName($name)
    {
    	$this->name = $name;
    }

    public function getName()
    {
    	return $this->name;
    }
}
```

Da notare che abbiamo definito due funzioni: addTags e removeTags. Queste devono essere sempre definite e vengono chiamate automaticamente dall'hydrator di Doctrine quando si tratta di collezioni.
S potrebbe pensare che questo sia eccessivo, e chiedersi perché non si possa semplicemente definire una funzione `setTags` per sostituire la vecchia collezione dalla nuova:

```php
public function setTags(Collection $tags)
{
	$this->tags = $tags;
}
```

Ma questo sarebbe sbagliato, perché le collezioni Doctrine non devono essere scambiate, soprattutto perché le collezioni sono gestite da
un ObjectManager, quindi non devono essere sostituite da una nuova istanza.

Ancora una volta, possono verificarsi due casi: tags già esistenti o no.

#####Entità esistente nell'associazione

Quando esiste già un'entità nell'associazione, quello che dobbiamo fare è semplicemente dare gli identificatori delle entità:

```php
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$blogPost = new BlogPost();
$data = array(
	'title' => 'The best blog post in the world !',
	'tags'  => array(
		array('id' => 3), // add tag whose id is 3
		array('id' => 8)  // also add tag whose id is 8
	)
);

$blogPost = $hydrator->hydrate($data, $blogPost);

echo $blogPost->getTitle(); // prints "The best blog post in the world !"
echo count($blogPost->getTags()); // prints 2
```

**NOTE** : ancora una volta, questo:

```php
$data = array(
	'title' => 'The best blog post in the world !',
	'tags'  => array(
		array('id' => 3), // add tag whose id is 3
		array('id' => 8)  // also add tag whose id is 8
	)
);
```

può essere scritto:

```php
$data = array(
	'title' => 'The best blog post in the world !',
	'tags'  => array(3, 8)
);
```

#####Entità non esistente nell'associazione

Se entità dell'associazione non esiste, c'è solo bisogno di dare l'oggetto dato:

```php
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($entityManager);
$blogPost = new BlogPost();

$tags = array();

$tag1 = new Tag();
$tag1->setName('PHP');
$tags[] = $tag1;

$tag2 = new Tag();
$tag2->setName('STL');
$tags[] = $tag2;

$data = array(
	'title' => 'The best blog post in the world !',
	'tags'  => $tags // Note that you can mix integers and entities without any problem
);

$blogPost = $hydrator->hydrate($data, $blogPost);

echo $blogPost->getTitle(); // prints "The best blog post in the world !"
echo count($blogPost->getTags()); // prints 2
```

Per ottenere questo, è necessario modificare leggermente la mappatura, in modo che Doctrine possa persistere nuove entità sulle
associazioni (notare l'opzione cascade sulla associazione OneToMany):

```php

namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
class BlogPost
{
	/** .. */

    /**
     * @ORM\OneToMany(targetEntity="Application\Entity\Tag", mappedBy="blogPost", cascade={"persist"})
     */
	protected $tags;

	/** … */
}
```

#####Gestione dei valori nulli

Quando un valore nullo viene passato a un campo OneToOne o ManyToOne, per esempio:

```php
$data = array(
    'city' => null
);
```

L'hydrator verifica se il metodo setCity() sull'entità consente valori Null e agisce di conseguenza:

1. Se il metodo setCity () non consente valori Null cioè `function setCity(City $city)`, il null viene silenziosamente ignorato allora non verrà idratato.
2. Se il metodo setCity () consente valori Null cioè `function setCity(City $city = null)`, sarà idratato il valore null.
