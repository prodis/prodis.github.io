---
layout: post
title: Miss Elixir
permalink: miss-elixir
tags: elixir miss library package
---

[*Miss Elixir*](https://github.com/prodis/miss-elixir) library brings in a non-intrusive way some extra functions that, for
different reasons, are not part of the Elixir core.

<!-- more -->

## Motivation

After almost two years working full-time developing Elixir applications, I miss some functions in Elixir modules like `Kernel`,
`Map` and `String`.

Inevitably my team and I created some "infamous util modules", that in general each big application has, to fill the lack of some
functions.

Of course you might be thinking now: "Oh man! One more library with utility functions...". And maybe you are right, or maybe not. :)

Said that, I kindly dare you to take a look in the examples below and see some real code where the functions in
[*Miss Elixir*](https://hex.pm/packages/miss) were inspired. And after that, you make your conclusions if some of those functions
maybe should be part of the Elixir core in the future versions.

## Design decisions

Before jumping to the examples, I just want to make clear some design decisions in *Miss Elixir*:
- The order of the `Miss` namespace preceding the existing Elixir modules to be extended was made by intention. For example, `Miss.String`.
- The modules in *Miss Elixir* are not intended to be used with aliases. Always use the entire namespace to make explicit that module/function does not belong to Elixir core.
- None of the functions in *Miss Elixir* has the same name of functions present in the correspondent Elixir module.

## Examples

### Miss.String

In Erlang and Elixir concatenating binaries will copy the concatenated binaries into a new binary.  Every time you concatenate
binaries (`<>`) or use interpolation (`#{}`) you are making copies of those binaries.

To build a string, it is cheaper and more efficient to use IO lists to build the binary just once instead of concatenating along the way.

See the [Elixir IO Data documentation](https://hexdocs.pm/elixir/IO.html#module-io-data) for more information.

In the example below, the private function `build_href/3` uses [`IO.iodata_to_binary/1`](https://hexdocs.pm/elixir/IO.html#iodata_to_binary/1)
to build a string from four pieces of other strings and one integer value:

```elixir
defmodule LinkDecorator do
  ...

  @spec decorate(Link.t(), String.t(), integer()) :: Link.t()
  def decorate(%Link{type: :book} = link, url, link_id) do
    %{
      link
      | method: "POST",
        href: build_href(url, "book", link_id)
    }
  end

  def decorate(%Link{type: :pre_book} = link, url, link_id) do
    %{
      link
      | method: "GET",
        href: build_href(url, "pre-book", link_id)
    }
  end

  def decorate(%Link{} = link, _url, _link_id), do: link

  @spec build_href(String.t(), String.t(), integer()) :: String.t()
  defp build_href(url, path, link_id),
    do: IO.iodata_to_binary([url, "/", path, "?link_id=", link_id])

  ....
end
```

The same code using [`Miss.String.build/1`](https://hexdocs.pm/miss/Miss.String.html#build/1) becomes more clear and explicit you
are building a string:

```elixir
defmodule LinkDecorator do
  ...

  @spec build_href(String.t(), String.t(), integer()) :: String.t()
  defp build_href(url, path, link_id),
    do: Miss.String.build([url, "/", path, "?link_id=", link_id])

  ...
end
```

Or using [`Miss.String.build/5`](https://hexdocs.pm/miss/Miss.String.html#build/5) (or the other function overloads with less
paramters) when you have the control of the quantity of parameters to build:

```elixir
defmodule LinkDecorator do
  ...

  @spec build_href(String.t(), String.t(), integer()) :: String.t()
  defp build_href(url, path, link_id),
    do: Miss.String.build(url, "/", path, "?link_id=", link_id)

  ...
end
```

Another case is replacing the use of [`Enum.join/2`](https://hexdocs.pm/elixir/Enum.html#join/2) (that internally uses
[`IO.iodata_to_binary/1`](https://hexdocs.pm/elixir/IO.html#iodata_to_binary/1)) to join strings. See the implementation of the
private function `build_holder_name/1` in the module below:

```elixir
defmodule PaymentParams do
  ...

  @spec build(Payment.t()) :: map()
  def build(%Payment{
         billing: billing,
         credit_card: %CreditCard{
           expiration_month: expiration_month,
           expiration_year: expiration_year,
           transaction_id: transaction_id,
           type: type
         }
       }) do
    %{
      card: %{
        brand: CreditCardType.brand(type),
        expiration_month: expiration_month,
        expiration_year: expiration_year,
        holder_name: build_holder_name(billing),
        token: transaction_id
      },
      type: :card
    }
  end

  @spec build_holder_name(Billing.t()) :: String.t()
  defp build_holder_name(%Billing{first_name: first_name, last_name: last_name}),
    do: Enum.join([first_name, last_name], " ")

  ...
end
```

Now using [`Miss.String.build/3`](https://hexdocs.pm/miss/Miss.String.html#build/3):

```elixir
defmodule PaymentParams do
  ...

  @spec build_holder_name(Billing.t()) :: String.t()
  defp build_holder_name(%Billing{first_name: first_name, last_name: last_name}),
    do: Miss.String.build(first_name, " ", last_name)

  ...
end
```

### Miss.Kernel

Imagine you are piping some operations to build fields to create a struct. Using [`Kernel.struct/2`](https://hexdocs.pm/elixir/Kernel.html#struct/2)
it is necessary to assign the map to a variable before creating the struct:

```elixir
defmodule PreBookParams do
  ...

  defstruct [...]

  @spec build(Params.t(), Hotel.t(), Offer.t()) :: t()
  def build(%Params{} = params, %Hotel{} = hotel, %Offer{} = offer) do
    attrs =
      %{
        check_out: to_string(params.check_out),
        check_in: to_string(params.check_in),
        currency: params.currency_code,
        hotel_id: hotel.id,
        locale: params.language_code,
        rooms: RoomConfiguration.to_raw_rooms(params.rooms),
        user_ip: params.customer_ip
      }
      |> Map.merge(build_price(offer, params.currency_code))

    struct(__MODULE__, attrs)
  end

  ...
end
```

Using [`Miss.Kernel.struct_inverse/2`](https://hexdocs.pm/miss/Miss.Kernel.html#struct_inverse/2) the map can be piped when
creating the struct:


```elixir
defmodule PreBookParams do
  ...

  defstruct [...]

  @spec build(Params.t(), Hotel.t(), Offer.t()) :: t()
  def build(%Params{} = params, %Hotel{} = hotel, %Offer{} = offer) do
    %{
      check_out: to_string(params.check_out),
      check_in: to_string(params.check_in),
      currency: params.currency_code,
      hotel_id: hotel.id,
      locale: params.language_code,
      rooms: RoomConfiguration.to_raw_rooms(params.rooms),
      user_ip: params.customer_ip
    }
    |> Map.merge(build_price(offer, params.currency_code))
    |> Miss.Kernel.struct_inverse(__MODULE__)
  end

  ...
end
```

If you need to create a list of structs from an `Enumerable`, [`Miss.Kernel.struct_list/2`](https://hexdocs.pm/miss/Miss.Kernel.html#struct_list/2)
does this job for you.

```elixir
defmodule User do
  defstruct name: "User"
end

# Using a list of maps
iex> Miss.Kernel.struct_list(User, [
...>   %{name: "Akira"},
...>   %{name: "Fernando"}
...> ])
[
  %User{name: "Akira"},
  %User{name: "Fernando"}
]

# Using a list of keywords
iex> Miss.Kernel.struct_list(User, [
...>   [name: "Akira"],
...>   [name: "Fernando"]
...> ])
[
  %User{name: "Akira"},
  %User{name: "Fernando"}
]
```

Elixir [`Kernel`](https://hexdocs.pm/elixir/Kernel.html) has [`div/2`](https://hexdocs.pm/elixir/Kernel.html#div/2) to perform an
integer division and [`rem/2`](https://hexdocs.pm/elixir/Kernel.html#rem/2) to compute the remainder of an integer division, but
misses a function to do both at the same time.

[`Miss.Kernel.div_rem/2`](https://hexdocs.pm/miss/Miss.Kernel.html#div_rem/2) to the rescue:

```elixir
iex> Miss.Kernel.div_rem(5, 2)
{2, 1}
```

### Miss.Map

The next code example was not extracted from a real application, but it is clear and didactic enough to show how to convert a
struct to a map going through all nested structs using [`Miss.Map.from_nested_struct/2`](https://hexdocs.pm/miss/Miss.Map.html#from_nested_struct/2).

Given the following `Post` struct with author and comments:

```elixir
defmodule Post do
  defstruct [:title, :text, :date, :author, comments: []]
end

defmodule Author do
  defstruct [:id, :name]
end

defmodule Comment do
  defstruct [:text]
end

post = %Post{
  title: "My post",
  text: "Something really interesting",
  date: ~D[2010-09-01],
  author: %Author{
    id: 1234,
    name: "Pedro Bonamides"
  },
  comments: [
    %Comment{text: "Comment one"},
    %Comment{text: "Comment two"}
  ]
}
```

Using [`Map.from_struct/1`](https://hexdocs.pm/elixir/Map.html#from_struct/1) only the root struct will be converted:

```elixir
iex> Map.from_struct(post)
%{
  title: "My post",
  text: "Something really interesting",
  date: ~D[2010-09-01],
  author: %Author{
    id: 1234,
    name: "Pedro Bonamides"
  },
  comments: [
    %Comment{text: "Comment one"},
    %Comment{text: "Comment two"}
  ]
}
```

Using [`Miss.Map.from_nested_struct/2`](https://hexdocs.pm/miss/Miss.Map.html#from_nested_struct/2) all the nested structs are
converted:

```elixir
iex> Miss.Map.from_nested_struct(post)
%{
  title: "My post",
  text: "Something really interesting",
  date: %{
    calendar: Calendar.ISO,
    day: 1,
    month: 9,
    year: 2010
  },
  author: %{
    id: 1234,
    name: "Pedro Bonamides"
  },
  comments: [
    %{text: "Comment one"},
    %{text: "Comment two"}
  ]
}
```

But wait, there is something weird. The [`Date`](https://hexdocs.pm/elixir/Date.html) struct was also converted to a map, that it
is something not much useful.

To solve this situation, you can skip the conversion of any struct.

```elixir
iex> Miss.Map.from_nested_struct(post, [{Date, :skip}])
%{
  title: "My post",
  text: "Something really interesting",
  date: ~D[2010-09-01],
  author: %{
    id: 1234,
    name: "Pedro Bonamides"
  },
  comments: [
    %{text: "Comment one"},
    %{text: "Comment two"}
  ]
}
```

Or you can provide a function to transform a struct instead of converting to a map:

```elixir
iex> Miss.Map.from_nested_struct(post, [
...>   {Date, &to_string/1},
...>   {Comment, fn %Comment{text: text} -> text end}
...> ])
%{
  title: "My post",
  text: "Something really interesting",
  date: "2010-09-01",
  author: %{
    id: 1234,
    name: "Pedro Bonamides"
  },
  comments: [
    "Comment one",
    "Comment two"
  ]
}
```

### Miss.List

Last but not least, [`Miss.List.intersection/2`](https://hexdocs.pm/miss/Miss.List.html#intersection/2) returns a list containing
only the elements in common of the two given lists.

```elixir
iex> Miss.List.intersection([1, 2, 3, 4, 5], [3, 4, 5, 6, 7])
[3, 4, 5]
```

## Conclusion

I hope some of the functions in [*Miss Elixir*](https://hex.pm/packages/miss) can be helpful to your Elixir applications as they
are for mine.

There are more a couple of functions not mentioned in this post. Check the [full documentation](https://hexdocs.pm/miss).

And how about you, what function do you miss in Elixir? Of course, [contributions](https://github.com/prodis/miss-elixir),
suggestions and comments are welcome. :)
