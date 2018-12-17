# How to Encrypt Existing Data

When you adopt Cloak in an existing project, you often need to encrypt fields
that were not previously encrypted. This guide will help you through the
process, step by step.

## Create a Vault

Make sure you've set up (and configured!) a vault as described in the
`Cloak.Vault` documentation and [the installation guide](install.html).

## Create Local Ecto Types

Create local Ecto types for each kind of field you want to encrypt, as
described in the `Cloak.Vault` documentation and [the installation
guide](install.html).

## Add `encrypted_*` Fields

You'll need to create a duplicate version of each field you want to encrypt,
perhaps with the `encrypted_` prefix.

    alter table(:users) do
      add :encrypted_name, :binary
      add :encrypted_metadata, :binary
    end

Then, add the new fields to your schema. Make sure to keep them up to
date with changes using a temporary `put_encrypted_fields` function in
your changeset.

    defmodule MyApp.Accounts.User do
      use Ecto.Schema

      import Ecto.Changeset

      schema "users" do
        field :name, :string
        field :encrypted_name, MyApp.Encrypted.Binary
        field :metadata, :map
        field :encrypted_metadata, MyApp.Encrypted.Map
      end

      @doc false
      def changeset(struct, attrs \\ %{}) do
        struct
        |> cast(attrs, [:name, :metadata])
        |> put_encrypted_fields()
      end

      # Temporary function during the migration process, to ensure
      # that all changes are copied over to the new fields
      defp put_encrypted_fields(changeset) do
        changeset
        |> put_change(:encrypted_name, get_field(changeset, :name))
        |> put_change(:encrypted_metadata, get_field(changeset, :metadata))
      end
    end

## Copy Data to New Fields

In a custom mix task or the IEx console, run the following:

    MyApp.Accounts.User
    |> MyApp.Repo.all()
    |> Enum.map(fn user ->
      user
      |> Ecto.Changeset.change(%{
           encrypted_name: user.name,
           encrypted_metadata: user.metadata
         })
      |> MyApp.Repo.update!()
    end)

## Remove Original Fields

Once you're confident that all the data has migrated successfully to the new
fields, you can write a migration to remove the old, unencrypted fields.

    alter table(:users) do
      remove :name
      remove :metadata
    end

    rename table(:users), :encrypted_name, to: :name
    rename table(:users), :encrypted_metadata, to: :metadata

You can also remove the temporary `put_encrypted_fields/1` function, leaving
your schema like this:

    defmodule MyApp.Accounts.User do
      use Ecto.Schema

      import Ecto.Changeset

      schema "users" do
        field :name, MyApp.Encrypted.Binary
        field :metadata, MyApp.Encrypted.Map
      end

      @doc false
      def changeset(struct, attrs \\ %{}) do
        struct
        |> cast(attrs, [:name, :metadata])
      end
    end
