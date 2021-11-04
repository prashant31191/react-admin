---
layout: default
title: "Understanding The Show View"
---

## From A Manual Show View To React-Admin Components

The Show view is the simplest view in an admin: it displays a single record. You've probably developed it a dozen times, and in fact you don't need react-admin to build, say, a book show view:

```jsx
import { useParams } from 'react-router-dom';
import { useGetOne } from 'react-admin';
import { Card, Stack, Typography } from '@mui/material';

/**
 * Fetch a book from the API and display it
 */
const BookShow = () => {
    const { id } = useParams(); // this component is rendered in the /books/:id path
    const { data, loading, error } = useGetOne('books', id);
    if (loading) { return <Loading />; }
    if (error) { return <Error />; }
    return (
        <Card>
            <Stack spacing={1}>
                <div>
                    <Typography variant="caption" display="block">Title</Typography>
                    <Typography variant="body2">{data.title}</Typography>
                </div>
                <div>
                    <Typography variant="caption" display="block">Publication Date</Typography>
                    <Typography variant="body2">{new Date(data.published_at).toDateString()}</Typography>
                </div>
            </Stack>
        </Card>
    );
};
```

You can pass this `BookShow` component as the `show` prop of the `<Resource name="books" />`, and react-admin will render it on the `/books/:id/show` path.

This example uses the `useGetOne` hook instead of `fetch` because `useGetOne` already contains the authentication and request state logic. But you could totally write a Show view with `fetch`.

When you build Show views like the one above, you have to repeat quite a lot of code for each field. React-admin Field components can help avoid that repetition. The following example leverages the `<Labeled>`, `<TextField>`, and `<DateField>` components in tha purpose:

```jsx
import { useParams } from 'react-router-dom';
import { useGetOne, Labeled, TextField, DateField } from 'react-admin';
import { Card, Stack } from '@mui/material';

const BookShow = () => {
    const { id } = useParams();
    const { data, loading, error } = useGetOne('books', id);
    if (loading) { return <Loading />; }
    if (error) { return <Error />; }
    return (
        <Card>
            <Stack spacing={1}>
                <Labeled label="Title">
                    <TextField source="title" record={data} />
                </Labeled>
                <Labeled label="Publication Date">
                    <DateField source="published_at" record={data} />
                </Labeled>
            </Stack>
        </Card>
    );
};
```

Field components require a `record` to render, but they can grab it from a `RecordContext` instead of the `record` prop. This allows to reduce even more the amount of code you need to write for each field.

```jsx
import { useParams } from 'react-router-dom';
import { useGetOne, RecordContextProvider, Labeled, TextField, DateField } from 'react-admin';
import { Card, Stack } from '@mui/material';

const BookShow = () => {
    const { id } = useParams();
    const { data, loading, error } = useGetOne('books', id);
    if (loading) { return <Loading />; }
    if (error) { return <Error />; }
    return (
        <RecordContextProvider value={data}>
            <Card>
                <Stack spacing={1}>
                    <Labeled label="Title">
                        <TextField source="title" />
                    </Labeled>
                    <Labeled label="Publication Date">
                        <DateField source="published_at" />
                    </Labeled>
                </Stack>
            </Card>
        </RecordContextProvider>
    );
};
```

Displaying a stack of fields with a label in a Card is such a common task that react-admin provides a helper component for that. It's called `<SimpleShowLayout>`:

```jsx
import { useParams } from 'react-router-dom';
import { useGetOne, RecordContextProvider, SimpleShowLayout, TextField, DateField } from 'react-admin';

const BookShow = () => {
    const { id } = useParams();
    const { data, loading, error } = useGetOne('books', id);
    if (loading) { return <Loading />; }
    if (error) { return <Error />; }
    return (
        <RecordContextProvider value={data}>
            <SimpleShowLayout>
                <TextField label="Title" source="title" />
                <DateField label="Publication Date" source="published_at" />
            </SimpleShowLayout>
        </RecordContextProvider>
    );
};
```

The initial logic that fetches the record from the API and places it in a context is also common, and react-admin exposes the `<Show>` component to do it. So the example can be further simplified to the following:

```jsx
import { Show, SimpleShowLayout, TextField, DateField } from 'react-admin';

const BookShow = () => (
    <Show resource="books">
        <SimpleShowLayout>
            <TextField label="Title" source="title" />
            <DateField label="Publication Date" source="published_at" />
        </SimpleShowLayout>
    </Show>
);
```

And since the `<BookShow>` component is designed to be used within a `<Resource>`, there is no need to specify the `resource` prop in `<Show>`. The `<Resource>` component creates a `<ResourceContext>` with the resource name ('books' in the above example), and the `<Show>` component can use this context value when the `resource` prop isn't set. So we can reduce the boilerplate code even more: 

```jsx
import { Show, SimpleShowLayout, TextField, DateField } from 'react-admin';

const BookShow = () => (
    <Show>
        <SimpleShowLayout>
            <TextField label="Title" source="title" />
            <DateField label="Publication Date" source="published_at" />
        </SimpleShowLayout>
    </Show>
);
```

And that's it! React-admin components are not magic, they are React commponents designed to let you focus on the business logic and avoid repetitive tasks.

**Tip**: Actually, `<Show>` does more than the code it replaces in the previous example: it redirects to the List view if the call to `useGetOne` returns an error, it sets the page title, and stores all the data it prepared in a `<ShowContext>`.

**Tip**: Don't mix up the `RecordContext`, which stores a Record (e.g. `{ id: '1', title: 'The Lord of the Rings' }`), and the `<ResourceContext>`, which stores a resource name (e.g. `'book'`).

## Accessing the Record

Using the `<Show>` component instead of calling `useGetOne` manually has one drawback: there is no longer a `data` object containing the fetched record. Instead, you have to access the record from the `<RecordContext>` using the `useRecordContext` hook. 

The following example illustrates the usage of this hook with a custom Field component displaying stars according to the book rating:

```jsx
import { Show, SimpleShowLayout, TextField, DateField, useRecordContext } from 'react-admin';
import StarIcon from '@mui/icons-material/Star';

const NbStarsField = () => {
    const record = useRecordContext();
    return <>
        {[...Array(record.rating)].map((_, index) => <StarIcon key={index} />)}
    </>;
};

const BookShow = () => (
    <Show>
        <SimpleShowLayout>
            <TextField label="Title" source="title" />
            <DateField label="Publication Date" source="published_at" />
            <NbStarsField label="Rating" />
        </SimpleShowLayout>
    </Show>
);
```

Sometimes you don't want to create a new component just to be able to use the `useRecordContext` hook. In these cases, you can use the `<WithRecord>` component, which is the render prop version of the hook:

```jsx
import { Show, SimpleShowLayout, TextField, DateField, WithRecord } from 'react-admin';
import StarIcon from '@mui/icons-material/Star';

const BookShow = () => (
    <Show>
        <SimpleShowLayout>
            <TextField label="Title" source="title" />
            <DateField label="Publication Date" source="published_at" />
            <WithRecord label="Rating" render={record => <>
                {[...Array(record.rating)].map((_, index) => <StarIcon key={index} />)}
            </>} />
        </SimpleShowLayout>
    </Show>
);
```

## Using a Tabbed Layout

When a Show view has to display a lot of fields, the `<SimpleShowLayout>` component ends up in very long page that are not user-friendly. You can use the `<TabbedShowLayout>` component instead, which is a variant of the `<SimpleShowLayout>` component that displays the fields in tabs. 

```jsx
import { Show, TabbedShowLayout, Tab, TextField, DateField, WithRecord } from 'react-admin';
import StarIcon from '@mui/icons-material/Star';
import FavoriteIcon from '@mui/icons-material/Favorite';
import PersonPinIcon from '@mui/icons-material/PersonPin';

const BookShow = () => (
    <Show>
        <TabbedShowLayout>
            <Tab label="Description" icon={<FavoriteIcon />}>
                <TextField label="Title" source="title" />
                <ReferenceField label="Author" source="author_id">
                    <TextField source="name" />
                </ReferenceField>
                <DateField label="Publication Date" source="published_at" />
            </Tab>
            <Tab label="User ratings" icon={<PersonPinIcon />}>
                <WithRecord label="Rating" render={record => <>
                    {[...Array(record.rating)].map((_, index) => <StarIcon key={index} />)}
                </>} />
                <DateField label="Last rating" source="last_rated_at" />
            </Tab>
        </TabbedShowLayout>
    </Show>
);
```

## Using a Custom Layout

In many cases, neither the `<SimpleShowLayout>` nor the `<TabbedShowLayout>` components are enough to display the fields you want. In these cases, pass your layout components directly as children of the `<Show>` component. As `<Show>` takes care of fetching the record and putting it in a `RecordContext`, you can use Field components directly. 

For instance, to display several fields in a single line, you can use material-ui's `<Grid>` component:

```jsx
import { Show, SimpleShowLayout, TextField, DateField, ReferenceField } from 'react-admin';
import { Card, CardContent, Grid } from '@mui/material';
import StarIcon from '@mui/icons-material/Star';

const BookShow = () => (
    <Show>
        <Card>
            <CardContent>
                <Grid container spacing={2}>
                    <Grid item xs={12} sm={6}>
                        <TextField label="Title" source="title" />
                    </Grid>
                    <Grid item xs={12} sm={6}>
                        <ReferenceField label="Author" source="author_id" reference="authors">
                            <TextField source="name" />
                        </ReferenceField>
                    </Grid>
                    <Grid item xs={12} sm={6}>
                        <DateField label="Publication Date" source="published_at" />
                    </Grid>
                    <Grid item xs={12} sm={6}>
                        <WithRecord label="Rating" render={record => <>
                            {[...Array(record.rating)].map((_, index) => <StarIcon key={index} />)}
                        </>} />
                    </Grid>
                </Grid>
            </CardContent>
        </Card>
    </Show>
);
```

You can also split the list of fields into two stacks, and use the `<SimpleShowLayout>` in the main panel:

```jsx
import { Show, SimpleShowLayout, TextField, DateField, WithRecord } from 'react-admin';
import StarIcon from '@mui/icons-material/Star';

const BookShow = () => (
    <Show>
        <Grid container spacing={2}>
            <Grid item xs={12} sm={8}>
                <SimpleShowLayout>
                    <TextField label="Title" source="title" />
                    <DateField label="Publication Date" source="published_at" />
                    <WithRecord label="Rating" render={record => <>
                        {[...Array(record.rating)].map((_, index) => <StarIcon key={index} />)}
                    </>} />
                </SimpleShowLayout>
            </Grid>
            <Grid item xs={12} sm={4}>
                <Typography>Details</Typography>
                <Stack spacing={1}>
                    <Labeled label="ISBN"><TextField source="isbn" /></Labeled>
                    <Labeled label="Last rating"><DateField source="last_rated_at" /></Labeled>
                </Stack>
            </Grid>
    </Show>
);
```