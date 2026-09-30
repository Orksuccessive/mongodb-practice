# MongoDB Schema Design — Embed vs Reference

## 1. Twitter Clone

### Main Collections

```text
users
tweets
follows
likes
comments
```

### `users`

```text
users
├── _id
├── username
├── email
├── displayName
├── bio
├── profileImage
├── createdAt
└── updatedAt
```

### `tweets`

```text
tweets
├── _id
├── authorId
├── content
├── media
├── createdAt
├── updatedAt
├── likeCount
├── commentCount
└── repostCount
```

### `comments`

```text
comments
├── _id
├── tweetId
├── authorId
├── content
└── createdAt
```

### `likes`

```text
likes
├── _id
├── tweetId
├── userId
└── createdAt
```

### `follows`

```text
follows
├── _id
├── followerId
├── followingId
└── createdAt
```

### Embed vs Reference Decisions

#### User → Tweets: Reference

**Decision:** Reference tweets using `authorId`.

**Reason:**

* A user can create thousands or millions of tweets.
* Tweets grow continuously.
* Embedding tweets would make the user document extremely large.
* Tweets need to be queried independently.

```text
User
  │
  └── authorId ──→ Tweet
```

#### Tweet → Author: Reference

Use `authorId` rather than embedding the complete user.

**Reason:**

* User information can change.
* The same user can author many tweets.
* Avoid duplicating username/profile information in every tweet.

#### Tweet → Comments: Reference

Comments should be stored separately.

**Reason:**

* A tweet can receive a very large number of comments.
* Comments are independently created and queried.
* Prevent the tweet document from growing indefinitely.

#### Tweet → Likes: Reference

Store likes separately.

**Reason:**

* A popular tweet can have millions of likes.
* Embedding every liker would create an unbounded array.
* Likes need to be queried independently.

For fast tweet display, maintain:

```text
likeCount
commentCount
```

inside the tweet as **denormalized counters**.

#### Follows: Separate Collection

Do not embed followers/following lists inside users.

**Reason:**

* A user can have millions of followers.
* The relationship is many-to-many.
* Following/unfollowing should be independently manageable.

### Twitter Clone — Summary

| Relationship     | Strategy          | Reason               |
| ---------------- | ----------------- | -------------------- |
| User → Tweets    | Reference         | Unbounded growth     |
| Tweet → Author   | Reference         | Avoid duplication    |
| Tweet → Comments | Reference         | Potentially huge     |
| Tweet → Likes    | Reference         | Potentially millions |
| User → Followers | Reference         | Many-to-many         |
| Like count       | Embed/denormalize | Fast reads           |
| Comment count    | Embed/denormalize | Fast reads           |

---

# 2. E-Commerce Application

### Main Collections

```text
users
products
categories
orders
reviews
```

### `users`

```text
users
├── _id
├── name
├── email
├── phone
├── addresses
├── createdAt
└── updatedAt
```

### `products`

```text
products
├── _id
├── name
├── description
├── price
├── sku
├── images
├── categoryId
├── stock
├── ratingAverage
├── reviewCount
└── createdAt
```

### `categories`

```text
categories
├── _id
├── name
├── slug
├── description
└── parentCategoryId
```

### `orders`

```text
orders
├── _id
├── userId
├── items[]
│   ├── productId
│   ├── productName
│   ├── quantity
│   ├── unitPrice
│   └── subtotal
├── shippingAddress
├── paymentStatus
├── orderStatus
├── totalAmount
└── createdAt
```

### `reviews`

```text
reviews
├── _id
├── productId
├── userId
├── rating
├── title
├── comment
└── createdAt
```

## Embed vs Reference Decisions

### User → Addresses: Embed

Addresses are good candidates for embedding.

**Reason:**

* Addresses are relatively small.
* They belong to the user.
* They are usually retrieved together with the user's account.
* A user normally has a limited number of addresses.

```text
User
└── addresses[]
```

### Product → Category: Reference

Use `categoryId`.

**Reason:**

* Many products can belong to the same category.
* Category information can change.
* Avoid duplicating category details across thousands of products.

```text
Product
└── categoryId ──→ Category
```

### Product → Reviews: Reference

Reviews should be a separate collection.

**Reason:**

* Products can accumulate a large number of reviews.
* Reviews are independently queried and paginated.
* Prevent product documents from becoming too large.

Maintain:

```text
ratingAverage
reviewCount
```

inside the product for fast product-list queries.

### Order → Order Items: Embed

Order items should be embedded inside the order.

```text
Order
└── items[]
```

**Reason:**

* Order items belong to that particular order.
* They are normally read together with the order.
* An order usually has a manageable number of items.
* The order should preserve what was purchased at that time.

### Important: Snapshot Product Information

An order item should contain:

```text
productId
productName
unitPrice
quantity
```

rather than relying only on the current product document.

**Why?**

Suppose:

```text
Product price today = $100
```

A customer purchased it for:

```text
$80
```

If the product later becomes $120, the historical order must still show:

```text
unitPrice = $80
```

Therefore, some product information is intentionally duplicated inside the order.

### Order → User: Reference

Use `userId`.

**Reason:**

* A user can have many orders.
* Orders need independent querying and pagination.
* Avoid embedding potentially thousands of orders inside a user document.

### E-Commerce — Summary

| Relationship             | Strategy             | Reason                   |
| ------------------------ | -------------------- | ------------------------ |
| User → Addresses         | Embed                | Small, bounded data      |
| Product → Category       | Reference            | Shared entity            |
| Product → Reviews        | Reference            | Potentially large        |
| Product → Rating summary | Embed/denormalize    | Fast reads               |
| Order → User             | Reference            | One-to-many              |
| Order → Items            | Embed                | Read together            |
| Order item → Product     | Reference + snapshot | Preserve historical data |
| Review → User            | Reference            | Shared user              |
| Review → Product         | Reference            | Shared product           |

---

# 3. Blog Application

### Main Collections

```text
users
posts
comments
tags
```

### `users`

```text
users
├── _id
├── username
├── email
├── displayName
├── profileImage
├── bio
└── createdAt
```

### `posts`

```text
posts
├── _id
├── authorId
├── title
├── slug
├── content
├── excerpt
├── tags[]
├── status
├── publishedAt
├── commentCount
└── createdAt
```

### `comments`

```text
comments
├── _id
├── postId
├── authorId
├── content
├── parentCommentId
└── createdAt
```

### `tags`

```text
tags
├── _id
├── name
├── slug
└── description
```

## Embed vs Reference Decisions

### Post → Author: Reference

Use:

```text
authorId
```

**Reason:**

* A user can write many posts.
* User profile information can change.
* Avoid duplicating user information across posts.

### Post → Tags: Usually Embed Tag IDs

Store:

```text
tags: [tagId1, tagId2, tagId3]
```

inside the post.

**Reason:**

* A post normally has a small, bounded number of tags.
* Tags are useful when querying/filtering posts.
* The relationship is many-to-many, but the number of tags per post is usually small.

The actual tag information remains in the `tags` collection.

```text
Post
├── tags[]
│   ├── tagId
│   ├── tagId
│   └── tagId
│
└──────────────→ Tags
```

### Post → Comments: Reference

Comments should be a separate collection.

**Reason:**

* A popular post can have thousands of comments.
* Comments require pagination.
* Comments are independently created and queried.
* Avoid unbounded arrays inside posts.

Maintain:

```text
commentCount
```

inside the post.

### Comment → Author: Reference

Use:

```text
authorId
```

because many comments can belong to the same user.

### Comment → Parent Comment: Reference

For nested/reply comments:

```text
parentCommentId
```

can point to another comment.

This avoids deeply nested comment arrays.

### Tags → Posts: Do Not Store Huge Post Arrays

Avoid:

```text
tag
└── posts: [thousands of post IDs]
```

Instead, query posts by their `tags` field.

This keeps the tag document small even when a tag becomes extremely popular.

### Blog — Summary

| Relationship         | Strategy          | Reason             |
| -------------------- | ----------------- | ------------------ |
| Post → Author        | Reference         | Shared user        |
| Post → Tags          | Embed tag IDs     | Small bounded list |
| Post → Comments      | Reference         | Potentially large  |
| Comment → Author     | Reference         | Shared user        |
| Comment → Parent     | Reference         | Avoid deep nesting |
| Post → Comment count | Embed/denormalize | Fast reads         |
| Tag → Posts          | Query from posts  | Avoid huge arrays  |

---

# General Embed vs Reference Rules

## Prefer Embedding When

Use embedding when:

* Data is small.
* Data is bounded.
* Child data belongs strongly to the parent.
* Child data is usually read together with the parent.
* Child data does not need independent lifecycle management.

Example:

```text
Order
└── items[]
```

---

## Prefer Referencing When

Use references when:

* Data can grow without a practical limit.
* The relationship is many-to-many.
* The child has an independent lifecycle.
* Data is frequently queried independently.
* The same entity is shared by many documents.

Example:

```text
User
   ↑
   │
Many Orders
```

---

# Key Design Principle

MongoDB schema design should be based primarily on **how the application reads and writes data**, not simply on normalization rules.

A practical approach is:

```text
Frequently read together
        ↓
     Embed

Independently queried
        ↓
    Reference

Potentially unbounded
        ↓
    Reference

Small + bounded
        ↓
     Embed

Historical data that must not change
        ↓
 Embed a snapshot
```

The most important exception is **denormalization for performance**. For example, `likeCount`, `commentCount`, `reviewCount`, and `ratingAverage` can be stored alongside the main document even though the underlying relationships are represented separately.
