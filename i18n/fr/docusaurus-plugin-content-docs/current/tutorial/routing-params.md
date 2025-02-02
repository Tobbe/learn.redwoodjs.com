---
id: routing-params
title: "Paramètres de Routes"
sidebar_label: "Paramètres de Routes"
---

Maintenant que notre page d'accueil liste l'ensemble des articles de notre blog, il est temps de créer une page présentant le détail d'un article. Commençons par générer une page et sa route associée:

    yarn rw g page BlogPost

> Remarquez que nous ne pouvons pas nommer cette page `Post` car une autre page homonyme a déjà été crée lors de notre précédente démonstration du scaffolding.

Pour chaque article listé sur la page d'accueil, ajoutons un lien qui pointe vers notre nouvelle page (sans oublier au passage les imports pour `Link` et `routes`):

```javascript {3,12}
// web/src/components/BlogPostsCell/BlogPostsCell.js

import { Link, routes } from "@redwoodjs/router";

// QUERY, Loading, Empty and Failure definitions...

export const Success = ({ posts }) => {
    return posts.map((post) => (
        <article key={post.id}>
            <header>
                <h2>
                    <Link to={routes.blogPost()}>{post.title}</Link>
                </h2>
            </header>
            <p>{post.body}</p>
            <div>Créé le: {post.createdAt}</div>
        </article>
    ));
};
```

Si vous cliquez sur le lien, vous deviez voir s'afficher un peu de texte issu de `BlogPostPage`. Mais ce dont nous avons vraiment besoin, c'est de pouvoir préciser _quel_ article nous souhaitons afficher. Ce que nous cherchons a obtenir en définitive, c'est une URL du type `/blog-post/1`. Pour cela, nous allons dire au routeur que notre url comporte une partie variable supplémentaire:

```html
// web/src/Routes.js

<Route path="/blog-post/{id}" page="{BlogPostPage}" name="blogPost" />
```

Notez l'ajout de `{id}` dans notre route. Redwood nomme ceci un _paramètre de route_. Ces paramètres de route signifie la chose suivante: "quelque soit la valeur à cette position, elle sera référencée par le nom utilisé entre les accolades".

Cool, cool, cool. Maintenant, nous devons donc construire un lien qui possède cet identifiant:

```html
// web/src/components/BlogPostsCell/BlogPostsCell.js

<Link to={routes.blogPost({ id: post.id })}>{post.title}</Link>
```

Pour les routes avec paramètres, un objet est attendu pour chaque paramètre. Si vous cliquez sur le lien d'un article, vous constaterez qu'en effet il pointe désormais vers `/blog-post/1` (ou `/blog-post/2`, etc... selon l'article).

### Utilisation des Paramètres

OK, donc l'identifiant se trouve bien dans l'URL. De quoi avons-nous besoin pour afficher un article spécifique ? On dirait bien que nous allons devoir récupérer les données depuis la base. Vous l'aurez compris, c'est le bon moment pour utiliser une Cell:

    yarn rw g cell BlogPost

Nous allons ensuite utiliser cette Cell dans notre page `BlogPostPage` (et pendant que nous y sommes, nous insèrerons notre page dans notre Layout `BlogLayout`):

```javascript
// web/src/pages/BlogPostPage/BlogPostPage.js

import BlogLayout from "src/layouts/BlogLayout";
import BlogPostCell from "src/components/BlogPostCell";

const BlogPostPage = () => {
    return (
        <BlogLayout>
            <BlogPostCell />
        </BlogLayout>
    );
};

export default BlogPostPage;
```

Maintenant, à l'intérieur de notre Cell, nous avons besoin d'accéder à ce paramètre de route `{id}` qui contient l'identifiant de notre article en base de données. Pour ce faire, mettons à jour la requête de façon à ce qu'elle accepte une variable en entrée. Modifions également le nom de la requête `blogPost` en `post`.

```javascript {4,5,7-9,20,21}
// web/src/components/BlogPostCell/BlogPostCell.js

export const QUERY = gql`
    query BlogPostQuery($id: Int!) {
        post(id: $id) {
            id
            title
            body
            createdAt
        }
    }
`;

export const Loading = () => <div>Loading...</div>;

export const Empty = () => <div>Empty</div>;

export const Failure = ({ error }) => <div>Error: {error.message}</div>;

export const Success = ({ post }) => {
    return JSON.stringify(post);
};
```

Okay, on approche du but! Ceci étant, d'où vient donc ce `$id`? Redwood a plus d'un tour dans son sac. Chaque fois que vous ajoutez un paramètre de route, ce paramètre est automatiquement accessible dans la page qui correspond. Ce qui signifie que vous pouvez modifier la page `BlogPostPage` de la façon suivante:

```javascript {3,6}
// web/src/pages/BlogPostPage/BlogPostPage.js

const BlogPostPage = ({ id }) => {
    return (
        <BlogLayout>
            <BlogPostCell id={id} />
        </BlogLayout>
    );
};
```

`id` existe déjà sans effort supplémentaire puisque nous avons nommé notre paramètre de route `{id}`. Merci Redwood! Mais comment se fait-il que cet `id` finisse par devenir un paramètre GraphQL `$id`? Si vous avez appris quoi que ce soit sur Redwood, vous devriez savoir qu'il va prendre soin de cela pour vous! Par défaut, chaque propriété que vous donnez à une Cell devient automatiquement un variable disponible pour une requête GraphQL. Et oui! C'est exact.

D'ailleurs on peut le prouver! Essayez de vous rendre à la page de détail d'un article dans le navigateur et — ahem.. Hmm:

![image](https://user-images.githubusercontent.com/300/75820346-096b9100-5d51-11ea-8f6e-53fda78d1ed5.png)

> Au passage le code d'erreur que vous voyez s'afficher provient de la section `Failure` de votre Cell!

Si vous examinez la console de votre navigateur, vous constaterez la présence d'une erreur GraphQL:

    [GraphQL error]: Message: Variable "$id" got invalid value "1";
      Expected type Int. Int cannot represent non-integer value: "1",
      Location: [object Object], Path: undefined

Il s'avère que les paramètres de route sont extraits des URL sous la forme de chaînes de caractères, et dans le cas présent GraphQL s'attend à recevoir un identifiant sous la forme d'un entier. Nous pourrions simplement utiliser la fonction javascript `parseInt()` afin de convertir notre paramètre de route vers un entier avant de le passer à `BlogPostCell`. Mais honnêtement, on peut faire bien mieux que ça!

### Paramètres de Route Typés

Et si vous aviez la possibilité de demander cette conversion directement dans le chemin de la route? Well, guess what: you can! Introducing **route param types**. It's as easy as adding `:Int` to our existing route param:

```html
// web/src/Routes.js

<Route path="/blog-post/{id:Int}" page="{BlogPostPage}" name="blogPost" />
```

Voilà! Non seulement vous allez convertir sans effort le paramètre `id` en un entier avant de la passer à votre Page, mais en bonus vous faîtes en sorte que la route n'applique que si `id` représente effectivement un entier, c'est à dire une suite de chiffres. Dans le cas contraire, le routeur essaiera d'autres routes. S'il ne s'en trouve aucune à s'appliquer, le routeur affichera la page `NotFoundPage`.

> **Que se passe-t-il si je veux passer d'autres propriétés à ma Cell dont je n'ai pas besoin dans la requête, mais qui me sont utile dans les composants Success/Loader/etc... ?**
> 
> Toutes les propriétés que vous donnez à votre Cell seront automatiquement disponibles pour ses composants internes. Seuls ceux qui se se trouvent dans la liste des variables GraphQL seront transmises à la requête. Vous avez ainsi le meilleur des deux mondes! Dans l'affichage de notre article ci-dessus, si vous désirez montrer par exemple un nombre au hasard (pour des raisons evidentes liées à ce didacticiel :D), il vous suffit de passer cette propriété à votre Cell:
> 
> ```javascript
<BlogPostCell id={id} rand={Math.random()} />
```

```javascript 

```javascript
Et ensuite vous la récupérez avec le résulat de la requête ans le composant (et même avec l'identifiant de l'article si vous le souhaitez): And get it, along with the query result (and even the original `id` if you want) in the component:

```javascript
export const Success = ({ post, id, rand }) => {
  //...
}
```

Merci Redwood!

### Afficher un Article

Maintenant, affichons le véritable article au lieu de simplement dumper le résultat de la requête. Il semble que ce soit l'endroit parfait pour utiliser un bon vieux composant puisque nous affichons les articles de façon identique (pour l'instant) à la fois sur la page d'accueil et sur la page de détail. Commençons par "Redwooder" un composant (j'ai juste inventé cette phrase) :

    yarn rw g component BlogPost

// web/src/components/BlogPost/BlogPost.js const BlogPost = () =&gt; { return (
        &lt;div&gt;
            &lt;h2&gt;{"BlogPost"}&lt;/h2&gt;
            &lt;p&gt;{"Find me in ./web/src/components/BlogPost/BlogPost.js"}&lt;/p&gt;
        &lt;/div&gt; ); }; export default BlogPost;

```javascript
// web/src/components/BlogPost/BlogPost.js

const BlogPost = () => {
  return (
    <div>
      <h2>{'BlogPost'}</h2>
      <p>{'Find me in ./web/src/components/BlogPost/BlogPost.js'}</p>
    </div>
  )
}

export default BlogPost
```

> Vous remarquerez peut-être que nous n'avons ici aucun `import` relatif à la librairie `React`. En réalité, nous (la "Redwood dev team") sommes un peu fatigués d'avoir à importer constamment les mêmes fichiers de la même manière... alors nous avons fait en sorte que Redwood le fasse pour nous, et donc pour vous!

L'exécution de cette commande créé le composant `BlogPost` dans le fichier `web/src/components/BlogPost/BlogPost.js`, accompagné de son fichier de test:

```javascript {3,5,7-14}
// web/src/components/BlogPost/BlogPost.js

import { Link, routes } from '@redwoodjs/router'

const BlogPost = ({ post }) => {
  return (
    <article>
      <header>
        <h2>
          <Link to={routes.blogPost({ id: post.id })}>{post.title}</Link>
        </h2>
      </header>
      <div>{post.body}</div>
    </article>
  )
}

export default BlogPost
```

Supprimons la partie de code qui affiche l'article dans `BlogPostCell`, et mettons la plutôt ici.

```javascript {3,8}
// web/src/components/BlogPostsCell/BlogPostsCell.js

import BlogPost from 'src/components/BlogPost'

// Loading, Empty, Failure...

export const Success = ({ posts }) => {
  return posts.map((post) => <BlogPost key={post.id} post={post} />)
}
```

```javascript {3,8}
// web/src/components/BlogPostCell/BlogPostCell.js

import BlogPost from 'src/components/BlogPost'

// Loading, Empty, Failure...

export const Success = ({ post }) => {
  return <BlogPost post={post} />
}
```

Et nous y sommes! Nous devrions maintenant pouvoir aller et venir à notre guise entre la page d'accueil et les articles.

> Si vous appréciez ce que vous venez de voir sur le routeur, vous pouvez en apprendre plus dans le [guide](https://redwoodjs.com/docs/redwood-router) qui lui est consacré.

### Résumé

Un petit état des lieux de ce que nous avons réalisé:

1. Création d'une nouvelle page pour afficher un article
2. Ajout d'une route prenant en char l'identifiant `id` d'un article sous la forme d'un paramètre de route
3. Création d'une Cell permettant de récupérer et afficher un article
4. Constat de la capacité de Redwood à vous mettre de bonne humeur en vous donnant accès à `id` là où vous en avez besoin tout en le convertissant au format numérique à la volée
5. Transformation de l'affichage d'un article en un composant React classique pouvant être partagé à plusieurs endroits dans l'interface (en l'espèce dans la page d'accueil et la page de détail)

