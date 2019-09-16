# Pluggy

Eftersom Servy innehåller en del svårfixade buggar byggde jag en skelettapplikation baserat på [Plug](https://hex.pm/packages/plug) och [Cowboy](https://github.com/ninenines/cowboy). Plug & Cowboy är (som nämndes i de senare Servy-filmerna)  de ramverk som i stort sett alla Elixir-webbramverk är baserade på. 

Pluggy följer samma generella upplägg som Servy, men med andra komponenter.

## Konfigurering och installation av dependencies

### Docker

Vi kommer köra vår Postgres databas-server i en docker.

Skapa ett konto på [https://hub.docker.com/](https://hub.docker.com/) och ladda ner och installera [docker-desktop för mac](https://hub.docker.com/editions/community/docker-ce-desktop-mac).

OBS: Om du startar om din dator måste du eventuellt starta docker-desktop-applikationen igen (beroende på vilka inställningar du valde när du installerade den).

### Postgres

#### Installation

För att hämta docker-imagen från docker hub: `docker pull postgres`

#### Starta

När docker-imagen är nedladad kan du starta containern genom

`docker run --rm   --name pg-docker -e POSTGRES_PASSWORD=docker -d -p 5432:5432 -v $HOME/docker/volumes/postgres:/var/lib/postgresql/data  postgres`

Följande växlar skickades till `docker run`:

* `--rm*`: Tar bort containern när vi är klara med den. Detta sparar diskutrymme (speciellt om vi kör många olika containers)

- `-- name`: namnet på containern. Används för att identifera den. Bra att veta om man t.ex måste stoppa containern.
- `-e`: Skicka vidare miljövariablen `POSTGRES_PASSWORD` med värdet `docker` till containern. Detta sätter superuser lösenordet för Postgres. 
- `-d`: Startar containern i bakgrunden (låter oss fortsätta skriva nya kommandon i samma terminalfönster)
- `-p`: Kopplar port 5432 på localhost till port 5432 i containern. Port 5432 är Postgres standardport, vilket innebär att vi inte behöver konfigurera eventuella verktyg för att prata med postgres mer än absolut nödvändigt.
- `-v`: Mounta `/var/lib/postgresql/data` i containern till `$HOME/docker/volumes/postgres` på host-datorn. Detta så att postgresdatabasen överlever även när containern tas bort.

#### PSQL

PSQL är ett program som låter oss koppla upp mot postgres-databas-servrar.

Installation: 

````zsh
brew install libpq
echo 'export PATH="/usr/local/opt/libpq/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
````

Koppla upp mot databasservern:

````zsh
psql -h localhost -U postgres -d postgres
````

Följande växlar skickades till psql:

* `-h`: Host (kan även vara en ip-adress eller hostname)
* `-U`: User- namnet på användaren vi vill logga in som
* `-d`Database - namnet på databasen vi vill koppla upp oss mot

Ange sen lösenordet (`docker`) eller vad som sattes i `docker run` enligt ovan.

I psql kan du använda `help` eller `\?` för att få fram hjälp. `\q` stänger ner psql.

Du kan även använda vanliga SQL-kommandon som t.ex `CREATE TABLE ...` eller `SELECT * FROM ...`

`\d` listar tabellerna `\d+ tabellens-namn` visar tabellens schema.

### Seeding av databasen

I `lib/mix/seed.ex` finns ett mix-script för att nollställa och seeda databasen. Kör `mix seed` för att köra scriptet.

## Filer och arkitektur

### mix.exs

```elixir
defmodule Pluggy.MixProject do
  use Mix.Project

  def project do
    [
      app: :pluggy, #eller er applikations namn
      version: "0.1.0",
      elixir: "~> 1.9.1",
      start_permanent: Mix.env() == :prod,
      deps: deps()
    ]
  end

  # Run "mix help compile.app" to learn about applications.
  def application do
    [
      applications: [:cowboy, :plug, :postgrex], #cowboy och plug måste startas
      extra_applications: [:logger],
      mod: {Pluggy, []} #och även själva applikationen själv.
    ]
  end

  # Run "mix help deps" to learn about dependencies.
  def deps do
    [
      {:plug_cowboy, "~> 2.0"},
      {:postgrex, "~> 0.13.5"},
      {:poolboy, "1.5.1"},
      {:bcrypt_elixir, "~> 1.0"}, #för hashning av lösenord
      {:slime, "~> 1.2"} #om man vill använda slime
    ]
  end
end
```

### config.exs

Oförändrad från servy (förutom rad 1)

```elixir
config :pluggy,
  db: [
    pool: DBConnection.Poolboy,
    pool_size: 20,
    host: "localhost", # or address
    database: "testdb",
    username: "testuser",
    password: "test"
  ]
```

### lib/pluggy.ex

Startar supervisorn som drar igång allt som behövs

```elixir
defmodule Pluggy do
  use Application

  def start(_type, _args) do
    IO.puts("Starting the application...")
    Pluggy.Supervisor.start_link()
  end
end
```

### lib/pluggy/supervisor.ex

Ser till cowboy och Pluggy.router dras igång och kopplas ihop.

```elixir
defmodule Pluggy.Supervisor do
  use Supervisor

  def start_link do
    IO.puts("Starting THE supervisor...")
    Supervisor.start_link(__MODULE__, :ok, name: __MODULE__)
  end

  def init(:ok) do
    children = [
      Plug.Adapters.Cowboy.child_spec(
        scheme: :http,
        plug: Pluggy.Router,
        options: [port: 3000]),
      {Postgrex, Keyword.put(Application.get_env(:pluggy, :db), :name, DB)}
    ]

    Supervisor.init(children, strategy: :one_for_one)
  end
end
```

### lib/pluggy/router.ex

Tar emot http-requests och skickar dem vidare till relevant controller

```elixir
defmodule Pluggy.Router do
  use Plug.Router

  alias Pluggy.FruitController
  alias Pluggy.UserController
  
  #sätter upp "priv/static" som "public-mapp" (lägg css och övrigt där)
  plug Plug.Static, at: "/", from: :pluggy 
  plug(:put_secret_key_base)

  plug(Plug.Session,
    store: :cookie,
    key: "_pluggy_session",
    encryption_salt: "cookie store encryption salt",
    signing_salt: "cookie store signing salt",
    key_length: 64,
    log: :debug,
    secret_key_base: "-- LONG STRING WITH AT LEAST 64 BYTES --"
  )

  plug(:fetch_session) #låter oss kontrollera sessionskakorna
  plug(Plug.Parsers, parsers: [:urlencoded, :multipart]) #formulär och uploads
  plug(:match)
  plug(:dispatch)

  
  get "/fruits",           do: FruitController.index(conn)
  get "/fruits/new",       do: FruitController.new(conn)
  get "/fruits/:id",       do: FruitController.show(conn, id)
  get "/fruits/:id/edit",  do: FruitController.edit(conn, id)
  
  post "/fruits",          do: FruitController.create(conn, conn.body_params)
 
  # should be put /fruits/:id, but put/patch/delete 
  # are not supported without hidden inputs
  post "/fruits/:id/edit", do: FruitController.update(conn, id, conn.body_params)

  # should be delete /fruits/:id, but put/patch/delete 
  # are not supported without hidden inputs
  post "/fruits/:id/destroy", do: FruitController.destroy(conn, id)


  post "/users/login",     do: UserController.login(conn, conn.body_params)
  post "/users/logout",    do: UserController.logout(conn)

  match _ do
    send_resp(conn, 404, "oops")
  end

  defp put_secret_key_base(conn, _) do
    put_in(
      conn.secret_key_base,
      "-- LONG STRING WITH AT LEAST 64 BYTES LONG STRING WITH AT LEAST 64 BYTES --"
    )
  end
end

```

### lib/pluggy/controllers

Tar emot data från routern, innehåller logik för validering, etc. Pratar med databasen via "models", och skickar relevant data till Template för rendering till webbläsaren

### lib/pluggy/controllers/fruit_controller.ex

```elixir
defmodule Pluggy.FruitController do
  
  alias Pluggy.Fruit
  alias Pluggy.User
  import Pluggy.Template, only: [render: 2]
  import Plug.Conn, only: [send_resp: 3]


  def index(conn) do

    #get user if logged in
    session_user = conn.private.plug_session["user_id"]
    current_user = case session_user do
      nil -> nil
      _   -> User.get(session_user)
    end

    #srender använder slime
    send_resp(conn, 200, srender("fruits/index", fruits: Fruit.all(), user: current_user))
  end

  #render använder eex
  def new(conn),      do: send_resp(conn, 200, render("fruits/new", []))
  def show(conn, id), do: send_resp(conn, 200, render("fruits/show", fruit: Fruit.get(id)))
  def edit(conn, id), do: send_resp(conn, 200, render("fruits/edit", fruit: Fruit.get(id)))
  
  def create(conn, params) do
    Fruit.create(params)
    
    #move uploaded file from tmp-folder 
    #(might want to first check that a file was uploaded)
    File.rename(params["file"].path, "priv/static/uploads/#{params["file"].filename}")
    redirect(conn, "/fruits")
  end

  def update(conn, id, params) do
    Fruit.update(id, params)
    redirect(conn, "/fruits")
  end

  def destroy(conn, id) do
    Fruit.delete(id)
    redirect(conn, "/fruits")
  end

  defp redirect(conn, url) do
    Plug.Conn.put_resp_header(conn, "location", url) |> send_resp(303, "")
  end

end

```

### lib/pluggy/controllers/user_controller.ex

```elixir
defmodule Pluggy.UserController do

	#import Pluggy.Template, only: [render: 2] #det här exemplet renderar inga templates
	import Plug.Conn, only: [send_resp: 3]

	def login(conn, params) do
		username = params["username"]
		password = params["pwd"]

		#Bör antagligen flytta SQL-anropet till user-model
		result =
		  Postgrex.query!(DB, "SELECT id, password_hash FROM users WHERE username = $1", [username],
		    pool: DBConnection.Poolboy
		  )

		case result.num_rows do
		  0 -> #no user with that username
		    redirect(conn, "/fruits")
		  _ -> #user with that username exists
		    [[id, password_hash]] = result.rows

		    #make sure password is correct
		    if Bcrypt.verify_pass(password, password_hash) do
		      Plug.Conn.put_session(conn, :user_id, id)
		      |>redirect("/fruits") #skicka vidare modifierad conn
		    else
		      redirect(conn, "/fruits")
		    end
		end
	end

	def logout(conn) do
		Plug.Conn.configure_session(conn, drop: true) #tömmer sessionen
		|> redirect("/fruits")
	end

	# def create(conn, params) do
	# 	#pseudocode
	# 	# in db table users with password_hash CHAR(60)
	# 	# hashed_password = Bcrypt.hash_pwd_salt(params["password"])
    #  	# Postgrex.query!(DB, "INSERT INTO users (username, password_hash) VALUES ($1, $2)", [params["username"], hashed_password], [pool: DBConnection.Poolboy])
    #  	# redirect(conn, "/fruits")
	# end

	defp redirect(conn, url), do: Plug.Conn.put_resp_header(conn, "location", url) |> send_resp(303, "")
end

```

### lib/pluggy/models

Pratar med databasen, skapar structs, kan även innehålla andra relevanta funktioner

### lib/pluggy/models/fruit.ex

```elixir
defmodule Pluggy.Fruit do
	defstruct(id: nil, name: "", tastiness: "")
	alias Pluggy.Fruit

	def all do
		Postgrex.query!(DB, "SELECT * FROM fruits", [], [pool: DBConnection.Poolboy]).rows
		|> to_struct_list
	end

	def get(id) do
		Postgrex.query!(DB, "SELECT * FROM fruits WHERE id = $1 LIMIT 1",  
		[String.to_integer(id)], [pool: DBConnection.Poolboy]).rows
		|> to_struct
	end

	def update(id, params) do
		name = params["name"]
		tastiness = String.to_integer(params["tastiness"])
		id = String.to_integer(id)
		Postgrex.query!(DB, "UPDATE fruits SET name = $1, tastiness = $2 WHERE id = $3",
		[name, tastiness, id], [pool: DBConnection.Poolboy])
	end

	def create(params) do
		name = params["name"]
		tastiness = String.to_integer(params["tastiness"])
		Postgrex.query!(DB, "INSERT INTO fruits (name, tastiness) VALUES ($1, $2)",
		[name, tastiness], [pool: DBConnection.Poolboy])	
	end

	def delete(id) do
		Postgrex.query!(DB, "DELETE FROM fruits WHERE id = $1", [String.to_integer(id)],
        [pool: DBConnection.Poolboy])	
	end

	def to_struct([[id, name, tastiness]]) do
		%Fruit{id: id, name: name, tastiness: tastiness}
	end

	def to_struct_list(rows) do
		for [id, name, tastiness] <- rows, do: %Fruit{id: id, name: name, tastiness: tastiness}
	end

end
```

### lib/pluggy/models/user.ex

```elixir
defmodule Pluggy.User do
	defstruct(id: nil, username: "")
	alias Pluggy.User

	def get(id) do
		Postgrex.query!(DB, "SELECT id, username FROM users WHERE id = $1 LIMIT 1", [id],
        pool: DBConnection.Poolboy
      ).rows |> to_struct
	end

	def to_struct([[id, username]]) do
		%User{id: id, username: username}
	end
end
```

### lib/pluggy/template.ex

Renderar slime eller eex-filer i templates-mappen. Gör det även möjligt att använda en layout-fil med gemensam html.

```elixir
defmodule Pluggy.Template do

  #slime
  def srender(file, data \\ [], layout \\ true) do
    {:ok, template} = File.read("templates/#{file}.slime")

    case layout do
      true ->
        {:ok, layout} = File.read("templates/layout.slime")
        Slime.render(layout, template: Slime.render(template, data))

      false ->
        Slime.render(template, data)
    end
  end

  #eex
  def render(file, data \\ [], layout \\ true) do
  	case layout do
    	true -> 
			EEx.eval_file("templates/layout.eex", template: EEx.eval_file("templates/#{file}.eex", data))
    	false -> 
    		EEx.eval_file("templates/#{file}.eex", data)
    end
  end
end
```

