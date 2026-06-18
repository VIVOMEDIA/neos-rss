# VIVOMEDIA.Rss

An RSS feed document node type for [Neos CMS](https://www.neos.io/).

Editors create an **RSS Feed** document anywhere in the content tree. The feed
collects descendant documents below a configurable starting point, optionally
filtered by node type, and renders them as RSS 2.0 XML. Each feed item can be
rendered with a dedicated, project-specific renderer.

- **Frontend:** the feed is served as XML under the document's URL with an
  `.xml` suffix (e.g. `/news.xml`) and sends an `application/rss+xml` header.
  The regular (`.html`) URL redirects to the `.xml` URL with a `303`.
- **Backend:** the document renders an HTML preview listing the feed items and
  the public feed URL.

## Requirements

- Neos `>= 8.3`

## Installation

```bash
composer require vivomedia/neos-rss
```

## Usage

Create an **RSS Feed** document in the content tree and configure it in the
inspector (group *RSS Feed*):

| Property          | Label                  | Description                                                                 |
| ----------------- | ---------------------- | --------------------------------------------------------------------------- |
| `rootNode`        | Ausgangspunkt          | Starting point of the feed. Descendants below this node are collected. *Required.* |
| `nodeTypeFilter`  | Seitentypen Filter     | Multi-select of allowed node types. Empty = **all** document types. The available options are empty by default and extended per project (see below). |
| `levels`          | Ebenen (0 = alle)      | How many levels below the starting point to include. `0` = unlimited.       |
| `limit`           | Max. Einträge (0 = alle) | Maximum number of items. `0` = unlimited. Default `20`.                    |
| `feedDescription` | Feed-Beschreibung      | `<description>` of the channel.                                             |

The channel title is the document's title; items are sorted by last publication
date (descending).

The feed is reachable at the document URL with an `.xml` suffix, e.g.
`https://example.com/news.xml`. Opening the document's HTML URL redirects there.

## Extending with custom node types

Out of the box the filter select box is empty, and every node type is rendered
with the default item renderer (title, link, publication date). Projects
typically do two things: offer specific node types in the filter, and provide a
dedicated rendering for them.

### 1. Add node types to the filter select box

Override `VIVOMEDIA.Rss:RssFeed` in your site package's `NodeTypes/` to populate
the `nodeTypeFilter` options:

```yaml
# NodeTypes/Override/RssFeed.Override.yaml
'VIVOMEDIA.Rss:RssFeed':
  properties:
    nodeTypeFilter:
      ui:
        inspector:
          editorOptions:
            values:
              'Acme.Site:NewsPage':
                label: 'News page'
                icon: 'fas fa-newspaper'
              'Acme.Site:BlogPost':
                label: 'Blog post'
                icon: 'fas fa-blog'
```

### 2. Provide a dedicated item renderer

For each node type the feed resolves a prototype named `<NodeType>.RssItem`. If that prototype exists (`Neos.Fusion:CanRender`), it is used;
otherwise it falls back to `VIVOMEDIA.Rss:DefaultRssItem`.

Define one in your project and compose the supplied presentational component,
`VIVOMEDIA.Rss:Component.RssItem`, which renders a valid `<item>`:

```fusion
prototype(Acme.Site:NewsPage.RssItem) < prototype(Neos.Fusion:Component) {
    title = ${q(node).property('title')}
    description = ${q(node).property('teaser')}
    pubDate = ${q(node).property('date') || q(node).property('_lastPublicationDateTime')}
    guid = ${node.identifier}
    link = Neos.Neos:NodeUri {
        node = ${node}
        format = 'html'
        absolute = true
    }

    renderer = afx`
        <VIVOMEDIA.Rss:Component.RssItem
                title={props.title}
                link={props.link}
                description={props.description}
                pubDate={props.pubDate}
                guid={props.guid}
        />
    `
}
```

#### `VIVOMEDIA.Rss:Component.RssItem` props

| Prop          | Description                                                                 |
| ------------- | --------------------------------------------------------------------------- |
| `title`       | Item title (wrapped in CDATA, tags stripped).                               |
| `link`        | Absolute item URL.                                                          |
| `description` | Item description (CDATA). Omitted when empty.                               |
| `pubDate`     | `DateTime`; formatted as RFC-822. Omitted when empty.                       |
| `guid`        | Unique identifier, rendered with `isPermaLink="false"`. Omitted when empty. |
| `content`     | AFX children — extra elements placed inside the `<item>`, e.g. `<content:encoded>` or an `<image>` element. |

To emit custom namespaced elements you usually also need to declare the XML
namespace on the `<rss>` element. Override the channel renderer's
`rssAttributes` in your project:

```fusion
prototype(VIVOMEDIA.Rss:RssFeed.Xml) {
    body.rssAttributes.'xmlns:content' = 'https://purl.org/rss/1.0/modules/content/'
}
```

