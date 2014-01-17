##Hydrator

Traduzione del seguente articolo [https://github.com/doctrine/DoctrineModule/edit/master/docs/hydrator.md](https://github.com/doctrine/DoctrineModule/blob/master/docs/hydrator.md)

Gli hydrators sono semplici oggetti che permettono di convertire una matrice di dati in un oggetto 
("hydrating") e di riconvertire un oggetto in un array ("extracting").

####Creare un hydrator

Per creare un Hydrator Doctrine abbimo bisogno soltanto di una cosa: un object manager (chiamato anche Entity Manager in Doctrine ORM)

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

Ora andiamo ad usare il hydrator Doctrine

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

### Collections strategy

Di default, le collezione di associazione avranno una strategia collegata che sarà chiamata durante la fase di idratamento (hydrating non so bene come tradurlo in IT) ed estrazione. Tutte queste strategie estendono dalla classe 
`DoctrineModule\Stdlib\Hydrator\Strategy\AbstractCollectionStrategy`.

DoctrineModule fornisce due tipi di strategie:

1. `DoctrineModule\Stdlib\Hydrator\Strategy\AllowRemoveByValue`: questa è la strategia di default, e rimuove i vecchi elementi che non sono nella nuova collezione.
2. `DoctrineModule\Stdlib\Hydrator\Strategy\AllowRemoveByReference`: questa è la strategia di default (in caso di settaggio byReference), e rimuove i vecchi elementi che non sono nella nuova collezionene.
3. `DoctrineModule\Stdlib\Hydrator\Strategy\DisallowRemoveByValue`: questa strategia non elimina i vecchi elementi, anche se questi non sono nella nuova collezione.
4. `DoctrineModule\Stdlib\Hydrator\Strategy\DisallowRemoveByReference`: questa strategia non elimina i vecchi elementi, anche se questi non sono nella nuova collezione.

Come conseguenza, quando usiamo `AllowRemove*`, dovremo definire sia l'adder (es. addTags) che il remover (es. removeTags).Mentre se usiamo la strategia `DisallowRemove*`, basterà definire l'adder, mentre il remover sarà opzionale (visto che gli elementi nono saranno mai rimossi).

La seguente tabella mostra le differenze tra le due strategie:

| Strategy | Initial collection | Submitted collection | Result |
| -------- | ------------------ | -------------------- | ------ |
| AllowRemove* | A, B  | B, C | B, C
| DisallowRemove* | A, B  | B, C | A, B, C

La differenza tra ByValue e ByReference è che quando si utilizzano strategie che finiscono per ByReference, Doctrine non si avvarrà 
dell'API pubblica della vostra entità (adder e remover) - non avremo nemmeno bisogno di definire i metodi -. Gli elementi saranno direttamente aggiunti e rimossi dalla collezione.

####Cambiare la strategia 

Cambiare la strategia per le collezioni è piuttosto semplice.

```php
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;
use DoctrineModule\Stdlib\Hydrator\Strategy;

$hydrator = new DoctrineHydrator($entityManager);
$hydrator->addStrategy('tags', new Strategy\DisallowRemoveByValue());
```

Si noti che è possibile anche aggiungere le strategie sui campi. 


###Per valore e per riferimento 

Per impostazione predefinita, l'Hydrator Doctrine opera per valore. Ciò significa che l'hydrator accederà e modificherà le proprietà attraverso l'API pubblica delle nostre entità (vale a dire, con getter e setter). Tuttavia, è possibile ignorare questa comportamento a lavorare per riferimento (vale a dire che l'hydrator accederà alle proprietà attraverso la Reflection API, e quindi bypasserà qualsiasi logica eventualmente inclusa nei setter / getter).

Per modificare il comportamento, basta passare il terzo parametro del costruttore come false:

```php
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($objectManager, false);
```

Per illustrare la differenza tra i due, facciamo una estrazione con la seguente entità:

```php

namespace Application\Entity;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
class SimpleEntity
{
    /**
     * @ORM\Column(type="string")
     */
	protected $foo;

	public function getFoo()
	{
		die();
	}

  	/** ... */
}
```

Utilizziamo il metodo predefinito, per valore:

```php
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($objectManager);
$object   = new SimpleEntity();
$object->setFoo('bar');

$data = $hydrator->extract($object);

echo $data['foo']; // never executed, because the script was killed when getter was accessed
```

Come possiamo vedere qui, lhydrator usa l'API pubblica (getFoo) per recuperare il valore. 

Tuttavia, se lo usiamo per riferimento:

```php
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;

$hydrator = new DoctrineHydrator($objectManager, false);
$object   = new SimpleEntity();
$object->setFoo('bar');

$data = $hydrator->extract($object);

echo $data['foo']; // prints 'bar'
```

Ora stampa "bar", il che mostra chiaramente che il getter non è stato chiamato.

###Un esempio completo utilizzando Zend\Form 

Ora che abbiamo capito come funziona l'hydrator, vediamo come integrarlo nel componente Form di Zend Framework 2. 
Abbiamo intenzione di utilizzare un semplice esempio con, ancora una volta, un'entità BlogPost ed una Tag. 

####Le entità 

Per prima cosa definiamo le (semplificate) entità:

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
     * @ORM\OneToMany(targetEntity="Application\Entity\Tag", mappedBy="blogPost", cascade={"persist"})
     */
	protected $tags;


	/**
	 * Never forget to initialize all your collections !
	 */
	public function __construct()
	{
		$this->tags = new ArrayCollection();
	}

	/**
	 * @return integer
	 */
    public function getId()
    {
   		return $this->id;
    }

	/**
	 * @param Collection $tags
	 */
	public function addTags(Collection $tags)
	{
		foreach ($tags as $tag) {
			$tag->setBlogPost($this);
			$this->tags->add($tag);
		}
	}

	/**
	 * @param Collection $tags
	 */
	public function removeTags(Collection $tags)
	{
		foreach ($tags as $tag) {
			$tag->setBlogPost(null);
			$this->tags->removeElement($tag);
		}
	}

	/**
	 * @return Collection
	 */
    public function getTags()
    {
    	return $this->tags;
    }
}
```

L'entità Tag:

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


	/**
	 * Get the id

	 * @return int
	 */
    public function getId()
    {
   		return $this->id;
    }

	/**
	 * Allow null to remove association
	 *
	 * @param BlogPost $blogPost
	 */
	public function setBlogPost(BlogPost $blogPost = null)
	{
		$this->blogPost = $blogPost;
	}

	/**
	 * @return BlogPost
	 */
    public function getBlogPost()
    {
    	return $this->blogPost;
    }

    /**
     * @param string $name
     */
    public function setName($name)
    {
    	$this->name = $name;
    }

    /**
     * @return string
     */
    public function getName()
    {
    	return $this->name;
    }
}
```

####I fieldsets 

Ora dobbiamo creare due fieldsets che mapperanno tali entità. Con Zend Framework 2, è una buona pratica creare 
un fieldset per ogni entità, al fine di poterli riutilizzare in molte forms. 

Ecco il fieldset per il Tag. Si noti che in questo esempio, ho aggiunto un input nascosto il cui nome è "id". questo è 
necessario per l'editing. La maggior parte del tempo, quando si crea il post sul blog per la prima volta, non esiste il tag, pertanto, l'id sarà vuoto. 
Tuttavia, quando si modifica il post sul blog, tutti i tags già esistono sul DB, e quindi l'input nascosto "id" avrà un valore. Ciò consente di modificare il nome di un tag modificando un'entità Tag esistente senza creare un nuovo tag (e rimozione del vecchio).

```php

namespace Application\Form;

use Application\Entity\Tag;
use Doctrine\Common\Persistence\ObjectManager;
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;
use Zend\Form\Fieldset;
use Zend\InputFilter\InputFilterProviderInterface;

class TagFieldset extends Fieldset implements InputFilterProviderInterface
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('tag');

        $this->setHydrator(new DoctrineHydrator($objectManager))
             ->setObject(new Tag());

		$this->add(array(
			'type' => 'Zend\Form\Element\Hidden',
			'name' => 'id'
		));

        $this->add(array(
            'type'    => 'Zend\Form\Element\Text',
            'name'    => 'name',
            'options' => array(
                'label' => 'Tag'
            )
        ));
    }

    public function getInputFilterSpecification()
    {
        return array(
            'id' => array(
            	'required' => false
            ),

            'name' => array(
                'required' => true
            )
        );
    }
}
```

Ed il fieldset BlogPost:

```php

namespace Application\Form;

use Application\Entity\BlogPost;
use Doctrine\Common\Persistence\ObjectManager;
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;
use Zend\Form\Fieldset;
use Zend\InputFilter\InputFilterProviderInterface;

class BlogPostFieldset extends Fieldset implements InputFilterProviderInterface
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('blog-post');

        $this->setHydrator(new DoctrineHydrator($objectManager))
             ->setObject(new BlogPost());

		$this->add(array(
			'type' => 'Zend\Form\Element\Text',
			'name' => 'title'
		));

		$tagFieldset = new TagFieldset($objectManager);
        $this->add(array(
            'type'    => 'Zend\Form\Element\Collection',
            'name'    => 'tags',
            'options' => array(
            	'count'           => 2,
                'target_element' => $tagFieldset
            )
        ));
    }

    public function getInputFilterSpecification()
    {
        return array(
            'title' => array(
            	'required' => true
            ),
        );
    }
}
```

Semplice e facile. Il blog post è solo un semplice fieldset con un  elemento di tipo ``Zend\Form\Element\Collection`` che rappresenta l'associazione ManyToOne. 

####Il modulo 

Ora che abbiamo creato il nostro fieldset, creeremo due moduli: un modulo per la creazione ed un modulo per l'aggiornamento. 
Il compito della form è quello di fare da collante tra i fieldsets. In questo semplice esempio, entrambe le forms sono esattamente uguali, 
ma in una vera e propria applicazione, è possibile modificare questo comportamento cambiando i gruppo di convalida (per esempio, 
si  potrebbe decidere di non consentire all'utente di modificare il titolo del post sul blog durante l'aggiornamento). 

Ecco pa fiorm di creazione:

```php
namespace Application\Form;

use Doctrine\Common\Persistence\ObjectManager;
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;
use Zend\Form\Form;

class CreateBlogPostForm extends Form
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('create-blog-post-form');

		// The form will hydrate an object of type "BlogPost"
        $this->setHydrator(new DoctrineHydrator($objectManager));

        // Add the user fieldset, and set it as the base fieldset
        $blogPostFieldset = new BlogPostFieldset($objectManager);
        $blogPostFieldset->setUseAsBaseFieldset(true);
        $this->add($blogPostFieldset);

        // … add CSRF and submit elements …

        // Optionally set your validation group here
    }
}
```

E quella di aggiornamento:

```php
namespace Application\Form;

use Doctrine\Common\Persistence\ObjectManager;
use DoctrineModule\Stdlib\Hydrator\DoctrineObject as DoctrineHydrator;
use Zend\Form\Form;

class UpdateBlogPostForm extends Form
{
    public function __construct(ObjectManager $objectManager)
    {
        parent::__construct('update-blog-post-form');

		// The form will hydrate an object of type "BlogPost"
        $this->setHydrator(new DoctrineHydrator($objectManager));

        // Add the user fieldset, and set it as the base fieldset
        $blogPostFieldset = new BlogPostFieldset($objectManager);
        $blogPostFieldset->setUseAsBaseFieldset(true);
        $this->add($blogPostFieldset);

        // … add CSRF and submit elements …

        // Optionally set your validation group here
    }
}
```

####I controllers

Ora abbiamo tutto. Creiamo i controllers. 

#####Creazione 

Nella createAction, creeremo un nuovo BlogPost e tutti i tags associati. Di conseguenza, gli IDs nascosti per i tags saranno da vuoti. 

Ecco l'azione per creare un nuovo post sul blog:

```php

public function createAction()
{
    // Get your ObjectManager from the ServiceManager
    $objectManager = $this->getServiceLocator()->get('Doctrine\ORM\EntityManager');

	// Create the form and inject the ObjectManager
	$form = new CreateBlogPostForm($objectManager);

	// Create a new, empty entity and bind it to the form
	$blogPost = new BlogPost();
	$form->bind($blogPost);

	if ($this->request->isPost()) {
		$form->setData($this->request->getPost());

		if ($form->isValid()) {
			$objectManager->persist($blogPost);
			$objectManager->flush();
		}
	}

	return array('form' => $form);
}
```

Il modulo di aggiornamento è simile, qui otterremo il post dal database invece di creare uno vuoto:

```php

public function editAction()
{
    // Get your ObjectManager from the ServiceManager
    $objectManager = $this->getServiceLocator()->get('Doctrine\ORM\EntityManager');

	// Create the form and inject the ObjectManager
	$form = new UpdateBlogPostForm($objectManager);

	// Create a new, empty entity and bind it to the form
	$blogPost = $this->userService->get($this->params('blogPost_id'));
	$form->bind($blogPost);

	if ($this->request->isPost()) {
		$form->setData($this->request->getPost());

		if ($form->isValid()) {
		    // Save the changes
		    $objectManager->flush();
		}
	}

	return array('form' => $form);
}
```

