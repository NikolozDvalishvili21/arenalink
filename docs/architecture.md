# ArenaLink: არქიტექტურა

Product Backlog: [backlog.md](backlog.md)

## არქიტექტურული სტილი

ArenaLink არის **მოდულური მონოლითი**: ერთი Next.js აპლიკაცია, რომელიც შიგნით რამდენიმე მოდულად არის დაყოფილი (Auth, Tournaments, Teams, Registration, Matches). მოდულები ერთმანეთს პირდაპირ იძახებენ.

სცენარად ავირჩიე D, საკუთარი იდეა, ამიტომ ლექტორთან შეთანხმება მჭირდება. A სცენარს (დატვირთვა და მასშტაბირება) ჰგავს იმით, რომ ტურნირის დღეს დატვირთვა იზრდება.

რატომ ეს სტილი:

1. **ერთი დეველოპერი, ერთი სემესტრი.** ერთ აპლიკაციას ერთხელ ვუშვებ და ერთი CI მაქვს. მიკროსერვისები ამ ზომის პროექტისთვის ზედმეტი სამუშაო იქნებოდა.
2. **რეგისტრაციას სიზუსტე სჭირდება.** ბოლო ადგილზე ორი გუნდი ერთად არ უნდა დარეგისტრირდეს (NFR-05). ერთ ბაზაში ამას ჩვეულებრივი ტრანზაქცია წყვეტს.
3. **Next.js უკვე ვიცი.** სამსახურშიც მას ვიყენებ, ამიტომ ახალი ფრეიმვორკის სწავლაზე დროს არ დავკარგავ.

## კომპონენტების დიაგრამა

```
ბრაუზერი -> Next.js (API) -> მოდულები (Auth, Tournaments, Teams, Registration, Matches) -> PostgreSQL
Registration -> გადახდის პროვაიდერი (TBC / BOG)
გადახდის პროვაიდერი -> Next.js (webhook)
```

 ბრაუზერი გვერდს Next.js-დან იღებს, მონაცემებს კი API route-ებიდან. API მოთხოვნას შესაბამის მოდულს გადასცემს, მოდული კი PostgreSQL-ს მიმართავს. გადახდისას Registration მოდული პროვაიდერს უკავშირდება, პროვაიდერი კი შედეგს webhook-ით აბრუნებს.

## Design Pattern-ები

- **Repository (backend).** `TournamentRepository`, `MatchRepository` და ა.შ. ბაზასთან მუშაობა ბიზნეს-ლოგიკისგან ცალკეა, ამიტომ ლოგიკის ტესტირება ბაზის გარეშე შემიძლია.
- **Strategy (backend).** `BracketGenerator` ინტერფეისს ახლა მხოლოდ `SingleElimination` ახორციელებს. თუ ოდესმე double elimination (US-14) გაკეთდება, ახალ კლასს დავამატებ და ძველს არ შევეხები.
- **Custom Hook (frontend).** `useBracket(tournamentId)`: მონაცემების წამოღება და განახლება hook-შია, კომპონენტი მხოლოდ ხატავს.

## API

API REST-ია, Next.js-ის route handler-ებით. ძირითადი ოპერაციები:

| მეთოდი | მისამართი | რას აკეთებს | Story |
|---|---|---|---|
| GET | `/api/tournaments` | ტურნირების სია (საჯარო) | US-02 |
| GET | `/api/tournaments/:id/bracket` | ბრეკეტი მატჩებითა და ანგარიშებით (საჯარო) | US-07 |
| GET | `/api/me/teams` | ჩემი გუნდები | US-03 |
| POST | `/api/tournaments/:id/registrations` | გუნდის რეგისტრაცია, ადგილი 10 წუთით ინახება | US-04 |
| POST | `/api/matches/:id/result` | მატჩის შედეგის გაგზავნა | US-06 |

შედეგის დასადასტურებლად ცალკე endpoint არ მჭირდება. პირველი კაპიტანი აგზავნის ანგარიშს და მატჩი „ელოდება დადასტურებას" სტატუსში გადადის. თუ მეორე კაპიტანი იგივე ანგარიშს აგზავნის, მატჩი იხურება, თუ სხვას, მატჩი „სადავო" ხდება.

შესვლა, ტურნირის შექმნა და სხვა ოპერაციებიც იქნება, აქ მხოლოდ მთავარი ხუთი ჩამოვწერე.

## ბაზა

ბაზად PostgreSQL ავირჩიე. მონაცემები ერთმანეთთან მჭიდროდ არის დაკავშირებული (გუნდი, რეგისტრაცია, გადახდა, მატჩი) და გადახდისთვის ტრანზაქციები მჭირდება, ამიტომ SQL უფრო მოხერხებულია.

სქემა (შემოკლებული):

```sql
CREATE TABLE users (
  id            SERIAL PRIMARY KEY,
  email         TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,
  role          TEXT NOT NULL            -- player, organizer, admin
);

CREATE TABLE teams (
  id         SERIAL PRIMARY KEY,
  name       TEXT UNIQUE NOT NULL,
  captain_id INT NOT NULL REFERENCES users(id)
);

CREATE TABLE team_members (
  team_id INT REFERENCES teams(id),
  user_id INT REFERENCES users(id),
  status  TEXT NOT NULL,                 -- invited, accepted
  PRIMARY KEY (team_id, user_id)
);

CREATE TABLE tournaments (
  id           SERIAL PRIMARY KEY,
  organizer_id INT NOT NULL REFERENCES users(id),
  name         TEXT NOT NULL,
  game         TEXT NOT NULL,
  starts_at    TIMESTAMPTZ NOT NULL,
  max_teams    INT NOT NULL CHECK (max_teams IN (4, 8, 16, 32)),
  entry_fee    NUMERIC(10,2) NOT NULL,
  status       TEXT NOT NULL             -- draft, open, closed, live, finished
);

CREATE TABLE registrations (
  id             SERIAL PRIMARY KEY,
  tournament_id  INT NOT NULL REFERENCES tournaments(id),
  team_id        INT NOT NULL REFERENCES teams(id),
  status         TEXT NOT NULL,          -- reserved, confirmed, expired
  reserved_until TIMESTAMPTZ,
  UNIQUE (tournament_id, team_id)
);

CREATE TABLE payments (
  id              SERIAL PRIMARY KEY,
  registration_id INT NOT NULL REFERENCES registrations(id),
  provider_ref    TEXT UNIQUE NOT NULL,
  amount          NUMERIC(10,2) NOT NULL,
  status          TEXT NOT NULL
);

CREATE TABLE matches (
  id            SERIAL PRIMARY KEY,
  tournament_id INT NOT NULL REFERENCES tournaments(id),
  round         INT NOT NULL,
  position      INT NOT NULL,
  team_a_id     INT REFERENCES teams(id),
  team_b_id     INT REFERENCES teams(id),
  score_a       INT,
  score_b       INT,
  status        TEXT NOT NULL,           -- scheduled, awaiting, disputed, done
  next_match_id INT REFERENCES matches(id)  -- სად გადადის გამარჯვებული
);

CREATE INDEX ON matches (tournament_id, round);
```

ადგილის დაჯავშნა (NFR-05) ერთ ტრანზაქციაშია: ჯერ ტურნირის სტრიქონს ვკეტავ (`SELECT ... FOR UPDATE`), ვითვლი `reserved` და `confirmed` რეგისტრაციებს და თუ ლიმიტი არ არის მიღწეული, ახალ რეგისტრაციას ვამატებ. ასე ორი გუნდი ერთ ადგილს ვერ დაიკავებს. ვადაგასულ `reserved` რეგისტრაციებს cron job ყოველ წუთში ათავისუფლებს.

## ტექნოლოგიები

- **Frontend და backend:** Next.js და TypeScript, ერთ პროექტში.
- **ბაზა:** PostgreSQL და Prisma.
- **ავტორიზაცია:** JWT httpOnly cookie-ში, პაროლები bcrypt-ით.
- **გადახდა:** TBC ან BOG სატესტო რეჟიმში.
- **ტესტები და CI:** Jest, k6 (დატვირთვის ტესტი) და GitHub Actions.

## რისკები

**ტურნირის დღის დატვირთვა.** ბევრმა ადამიანმა ერთად რომ გახსნას ბრეკეტი, ბაზა შეიძლება გადაიტვირთოს (NFR-01, NFR-02). ბრეკეტის query-სთვის ინდექსი მაქვს და დატვირთვას k6-ით წინასწარ გავტესტავ. ბრეკეტი WebSocket-ის ნაცვლად ყოველ 3 წამში ახლდება (polling), რაც უფრო მარტივია და NFR-03-ს აკმაყოფილებს. თუ ეს არ იკმარებს, კეშს ან WebSocket-ს მერე დავამატებ.

**გადახდა და ადგილი.** პროვაიდერის webhook შეიძლება დაგვიანდეს ან ორჯერ მოვიდეს. ადგილი 10 წუთით `reserved` სტატუსშია, `provider_ref` უნიკალურია, ამიტომ ერთი გადახდა ორჯერ არ ჩაითვლება, ვადაგასული ადგილი კი თავისუფლდება.

**სამუშაოს მოცულობა.** Must და Should ჯამში 56 ქულაა, რაც ერთი სემესტრისთვის ბევრია. ამიტომ MVP მხოლოდ Must story-ებია (37 ქულა), დანარჩენი დროის მიხედვით.
