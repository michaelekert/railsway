---
layout: post
title:  "Logowanie API z wykorzystaniem Ruby on Rails"
date:   2023-01-22
categories: api
---

Na wstępie należy wygenerować naszą aplikacje 

{% highlight ruby %}
rails new simple_api_auth --api
{% endhighlight %}

- —api oznacza że wygeneruje nam się aplikacja bez widoków i middlewarów

Następnie możemy przejść do wygenerowania modelu

{% highlight ruby %}
rails g model User email password_digest
{% endhighlight %}

- password_digest pozwala na wykonanie metody has_secures_password i zahashowanie hasła za pomocą bcrypt

{% highlight ruby %}
# models/user.rb

class User < ApplicationRecord
  has_secure_password
end
{% endhighlight %}

- email nie posiada weryfikacji i typ tej wartości to string

Następnie wykonujemy migracje 

{% highlight ruby %}
rails db:migrate
{% endhighlight %}

Przechodzimy do katalogu /models i w pliku user.rb dodajemy metodę has_secure_password, dzięki temu nasze hasło zahashowane za pomocą gem’a Bcrypt. Jednak aby go wykorzystać musimy go dodać do naszego pliku Gemfile.rb. Przy okazji dodamy już gem jwt (JSON Web Token), aby nie robić tego później 😉.

{% highlight ruby %}
# Gemfile.rb

gem "bcrypt", "~> 3.1.7"
gem "jwt"
{% endhighlight %}

Odpalamy terminal i wykonujemy poniższą komendę, aby pobrać nasze gemy 

{% highlight ruby %}
bundle install
{% endhighlight %}

Następnie przechodzimy do pisania pierwszego kodu. W katalogu /lib tworzymy plik o nazwie json_web_token.rb plik ten będzie odpowiadał za tworzenie i odszyfrowywanie tokenów (a.k żetonów 😂) JWT. 

{% highlight ruby %}
# lib/json_web_token.rb

class JsonWebToken

  SECRET = Rails.application.credentials.secret_key_base
  ENCRYPTION = 'none'

  def self.encode(payload, exp = 24.hours.from_now)
    payload[:exp] = exp.to_i
    JWT.encode(payload, SECRET)
  end

  def self.decode(token)
    body = JWT.decode(token, SECRET)[0]
    HashWithIndifferentAccess.new(body)
  rescue JWT::ExpiredSignature
    nil
  rescue
    nil
  end

end
{% endhighlight %}

Do tego celu stworzymy klasę, w której utworzymy dwie stałe. SECRET jest to losowy ciąg znaków wygenerowany przez Rails’y i warto dla bezpieczeństwa użyć właśnie go niż na twardo przypisać jakiś swój ciąg znaków (Jeśli chcesz zobaczyć swój SECRET możesz posłużyć się poniższą komendą. Przejdź do katalogu projektu w terminalu i wykonaj komendę:)

{% highlight ruby %}
EDITOR= nano rails credentials:edit
{% endhighlight %}

Zmienna na samym dole to właśnie nasz SECRET. Z kolei ENCRYPTION to stała która odpowiada za algorytm hashujący. W tym przypadku nie chcemy korzystać z żadnego. Następnie tworzymy metody klasy,  która koduje nasz JWT (JSON Web Token). Nasza metoda przyjmuje dwa parametry payload oraz exp(expire). Z czego z exp przekazujemy do naszego payload (payload skałada się z kilku oświadczeń, miedzy innymi z czasu wygaśnięcia, dlatego ustawiamy nasz parametr exp do payload). Na koniec kodujemy nasz żeton.

Czas teraz na odkodowywanie naszego żetonu po tym jak będziemy chcieli uwierzytelnić naszego użytkownika. Metoda klasy przyjmuje nasz żeton, a następnie przy pomocy naszego sekretu go dekoduje. Następnie tworzymy Hash’a z naszym odkodowanym żetonem. Na koniec jeśli nasz żeton wygaśnie to zwracamy nil, jak i w innym przypadku.

Teraz musimy dodać ścieżkę do naszego pliku. Przechodzimy do katalogu /config i w pliku application.rb dodajemy do klasy

{% highlight ruby %}
config.autoload_paths << Rails.root.join('lib')
{% endhighlight %}

Dzięki temu nasz plik będzie widziany przez Rails’y.

Teraz przechodzimy do najważniejszej części czyli głównej logiki biznesowej tego api. Na wstępie tworzymy katalog /services w katalogu /app. W katalogu /services tworzymy plik authenticate_user.rb. Plik ten będzie odpowiadał za uwierzytelnianie użytkownika.

{% highlight ruby %}
class AuthenticateUser < ApplicationService
  def initialize(email,password)
    @email = email
    @password = password
  end

  def call
    user = User.find_by(email: email)
    return user if user&.authenticate(password)
    errors.add(:user_auth, 'Invalid credientials')
  end
end
{% endhighlight %}

A więc tworzymy klasę AuthenticationUser, która dziedziczy z ApplicationService (będzie to omówione później). Na wstępie tworzymy konstruktor z parametrami email i password. Następnie tworzymy metodę call, której celem jest zwrócenie naszego User’a, jeśli podamy poprawne dane uwierzytelnianie przebiegnie pomyślnie (Uwierzytelnianie: Kto to ?, natomiast Autoryzacja: Czy masz dostęp ?), inaczej wyrzuca błąd. Teraz zabierzemy się za obsługę błędów. W katalogu /lib tworzymy tym razem plik o nazwie errors.rb

{% highlight ruby %}
# lib/errors.rb
class Errors < Hash
  def add(key, value, _opts = {})
    self[key] ||= []
    self[key] << value
    self[key].uniq!
  end
  def add_multiple_errors(errors_hash)
    errors_hash.each do |key,values|
      values.each { |value| add key, value }
    end
  end

  def each
    each_key do |field|
      self[field].each { |message| yield field, message }
    end
  end
end
{% endhighlight %}

Tworzymy klasę, która dziedziczy z klasy Hash. Dodajemy metodę add, która przyjmuję trzy argumenty:

- key, który jest nazwą błędu jaki dodajemy
- value jest jego opisem
- _opts są to dodatkowe argumenty, które możemy podać  w formie Hash’a

Metoda add odpowiada za “stworzenie” nowego błędu. Jeśli wartość instancji jest nie istnieje lub jest nil’em tworzy tablice. Następnie parametr value jest dodawany do tej tablicy. Ostatnia linijka pozbywa się duplikatów jeśli takie istnieją. Ostatnie dwie metody są jako dodatek, jednak nie będą one wykorzystywane w projekcie. Druga odpowiada za dodanie kilku błędów w postaci hasha (klucz to symbol lub string a wartość to tablica). Trzecia metoda pozwala na iteracji po tych elementach (Po zdefiniowaniu tej metody w moim edytorze pojawił się monit o tym że metoda jest nadpisana w hash.rb, wygląda na to, że w tamtym czasie iterowanie po elementach Hash’a nie było natywnie dostępne, jednak dzięki społeczności dodano taką metodę w aktualizacji 🙂). Teraz tworzymy plik o nazwie application_service.rb.

{% highlight ruby %}
class ApplicationService

  class << self
    def call(*arg)
      new(*arg).constructor
    end
  end

  attr_reader :result
  def constructor
    @result = call
    self
  end

  def success?
    !failure?
  end

  def failure?
    errors.any?
  end

  def call
    fail NotImplementedError unless defined?(super)
  end
end
{% endhighlight %}

A więc tworzymy klasę ApplicationService, która posiada metodę klasy o nazwie call. Przyjmuje ona argumenty (w naszym przypadku będzie to email i password) i tworzy nową instancje klasy ApplicationService następnie wykonuje na tej instancji metodę, która do zmiennej instancji result odwołuje się do metody call z AuthenticateUser, który zwraca na w zmiennej result naszego usera oraz zwraca sama siebie. Poniższe metody mają posłużyć nam w wykrywaniu błędów, a ostatnia służy do wyrzucenia błędu jeśli metoda call nie została stworzona. Teraz dodamy możliwość autoryzacji naszego użytkownika. Aby autoryzacja przebiegła pomyślnie musimy przekazać na token JWT w nagłówku. 

{% highlight ruby %}
class AuthorizeApiRequest < ApplicationService
  attr_reader :headers
  def initialize(headers = {})
    @headers = headers
  end

  def call
    user
  end

  private

  def user
    @user ||= User.find(decoded_auth_token[:user_id]) if decoded_auth_token
    @user || errors.add(:token, 'invalid token') && nil
  end

  def decoded_auth_token
    @decoded_auth_token ||= JsonWebToken.decode(http_auth_header)
  end

  def http_auth_header
    if headers['Authorization'].present?
      return headers['Authorization'].split(' ').last
    else
      errors.add(:token, 'missing token')
    end
    nil 
  end
end
{% endhighlight %}

A więc tworzymy plik o nazwie authorize_api_request.rb, w którym tworzymy klasę on tej samej nazwie. Tworzymy konstruktor nagłówka. Następnie tworzymy metodę call która zwraca nam metodę prywatną user. W tej metodzie zwracamy obiekt, który wyszukujemy po id użytkownika, natomiast jeśli nie znaleziono takiego id wyrzuca błąd (Nasz token jest zaszyfrowany więc aby zobaczyć czy token ten ma w sobie id użytkonika, którego szukamy musimy go odkodować. Odpowiada za to metoda decoded_auth_token). Wszystko wydawałoby się gotowe jednak w nagłówku http musimy pobrać tylko sam token, nie potrzebna nam jest cała zawartość (chodzi o to że pobieracjąć całość nasz token będzie zawierał do tego nazwe “Authorization”). Pomoże nam metoda http_auth_header, która sprawdza czy w nagłówku istnieje “Authorization”, jeśli tak zwraca nam ostatnią zawartość czyli interesujący nas token, w innym przypadku wyrzuca błąd. To by było wszystko, jeśli chodzi o service objects. Odpalamy terminal i tworzymy nasz kontroler sesji.

{% highlight ruby %}
rails g controller sessions create
{% endhighlight %}

Od razu możemy przejść do pliku routes.rb i dodać ścieżkę

{% highlight ruby %}
# routes.rb

Rails.application.routes.draw do
  post 'auth', to: 'sessions#create'
end
{% endhighlight %}

Przechodzimy teraz do kontrolera sesji i dodajemy poniższy kod.

{% highlight ruby %}
class SessionsController < ApplicationController
  skip_before_action :authenticate_request

  def create
    auth = AuthenticateUser.call(params[:email],params[:password])
    if auth.success?
      render json: { auth_token: auth.result}
    else
      render json: {errors: auth.errors}, status: :unauthorized
    end
  end
end
{% endhighlight %}

Na samej górze dodajemy aby nasz kontroler ominął akcje authenticate_request, która będzie nam potrzebna do autoryzacji. Metodę tą wykorzystamy w ApplicationController, który zajmie się autoryzacją. W metodzie create tworzymy zmienną, która wykonuje na klasie AuthenticateUser metodę call wraz z emailem i hasłem(Powyżej stworzyliśmy taką klasę, a w zasadzie cały program jest oparty na service objects. Główny plik application_service.rb wykonuję większość pracy przez co nasze kontrolery mają bardzo małą ilość kodu).

{% highlight ruby %}
class ApplicationService

  class << self
    def call(*arg)
      new(*arg).constructor
    end
  end

  attr_reader :result
  def constructor
    @result = call
    self
  end

  def success?
    !failure?
  end

  def failure?
    errors.any?
  end

  def errors
    @errors ||= Errors.new
  end
  def call
    fail NotImplementedError unless defined?(super)
  end
end
{% endhighlight %}

 Jeśli uwierzytelnianie przebiegło pomyślnie (metoda succes? jest właśnie z pliku application_service.rb) w odpowiedzi dostaniemy nasz token JWT jeśli natomiast podamy złe dane wyrzuci nam błąd. Teraz przechodzimy do autoryzacji. W ApplicationController tworzymy metodę authenticate_request.

{% highlight ruby %}
class ApplicationController < ActionController::API
  before_action :authenticate_request
  attr_reader :current_user

  private

  def authenticate_request
    auth = AuthorizeApiRequest.call(request.headers)
    @current_user = auth.result
    render json: {errors: auth.errors}, status: :unauthorized unless @current_user
  end
end
{% endhighlight %}

Musimy oczywiście dodać aby przed wykonaniem jakiejkolwiek akcji kontroler wykonał właśnie tą metodę. Dodajemy możliwość odwołania się do zmiennej instancji current_user. W naszej metodzie natomiast tworzymy zmienną auth w której to wykonujemy na klasie AuthorizeApiRequest wykonujemy metodę call z parametrami nagłówka http (a dokładnie chodzi o Authorization, w którym to znajdować się będzie nasz token). Wygląda na to, że udało nam się stworzyć system logowania do API. Jednak jest jeszczę , trochę rzeczy aby ulepszyć nasz program. W pliku, który już stworzyliśmy, a dokładnie authenticate_user.rb zmienimy trochę działanie metody call.

{% highlight ruby %}
class AuthenticateUser < ApplicationService
  def initialize(email,password)
    @email = email
    @password = password
  end
  private
  attr_accessor :email, :password

  def call
    JsonWebToken.encode(user_id: user.id) if user
  end

  def user
    user = User.find_by(email: email)
    return user if user&.authenticate(password)
    errors.add(:user_auth, 'Invalid credientials')
  end
end
{% endhighlight %}

chcemy, aby w odpowiedzi nasz program zamiast zwracał obiekt naszego użytkownika, zwrócił nam token, który jest zakodowany id tego użytkownika. Dobra teraz należałoby wynagrodzić użytkownika za poprawne uwierzytelnienie jak i autoryzacje. Tworzymy kontroler pages z index

 

{% highlight ruby %}
rails g controller pages index
{% endhighlight %}

A następnie zmieniamy ścieżkę na

{% highlight ruby %}
Rails.application.routes.draw do
  post 'auth', to: 'sessions#create'
	get 'pages', to: 'pages#index' 
end
{% endhighlight %}

Teraz wystarczy za pomocą curl lub Postmanem wykonać zapytanie z parametrami użytkownika (Musimy go wcześniej utworzyć w bazie danych), a w zamian dostaniemy token, którym będziemy się mogli dostać do [http://localhost:3000/pages](http://localhost:3000/pages) . 🙂

# Wnioski końcowe

Projekt pomimo, że nie jest dość złożony i pozwala na razie tylko dokonać logowania się do API, jeśli użytkownik znajduję się już w bazie danych. Jednak pomimo tego projekt ten nauczył mnie nowego podejścia do tworzenia aplikacji a mianowicie Service Objects. Na samym początku nie wiedziałem z czym to się je, jednak z coraz dłuższym pisaniem aplikacji zrozumiał zalety tego rozwiązania. Pomimo, że aplikacja była tworzona wraz z poradnikiem to napotkałem kilka problemów m. in: 

- Aby autoryzacja po tokenie działała musiałem ostatecznie pozbyć się zmiennej ENCRYPTION, po tym zabiegu token składał się z 3 części, zamiast 2

Przed

{% highlight ruby %}
# lib/json_web_token.rb

class JsonWebToken

  SECRET = Rails.application.credentials.secret_key_base
  ENCRYPTION = 'none'

  def self.encode(payload, exp = 24.hours.from_now)
    payload[:exp] = exp.to_i
    JWT.encode(payload, SECRET, ENCRYPTION)
  end

  def self.decode(token)
    body = JWT.decode(token, SECRET)
    HashWithIndifferentAccess.new(body)
  rescue JWT::ExpiredSignature
    nil
  rescue
    nil
  end

end
{% endhighlight %}

Po

{% highlight ruby %}
# lib/json_web_token.rb

class JsonWebToken

  SECRET = Rails.application.credentials.secret_key_base

  def self.encode(payload, exp = 24.hours.from_now)
    payload[:exp] = exp.to_i
    JWT.encode(payload, SECRET)
  end

  def self.decode(token)
    body = JWT.decode(token, SECRET)[0]
    HashWithIndifferentAccess.new(body)
  rescue JWT::ExpiredSignature
    nil
  rescue
    nil
  end

end
{% endhighlight %}

Pozostałe problemy były typowymi literówkami lub były powiązane z powyższym problemem.