# Inventronic

Electronic components management

![Parts UI](/img/parts.png)

![Project view](/img/project.png)

# Features

- Create electronic components (part, value, footprint, etc...).
- Create storage locations.
- Create projects that use certain parts.
- Print labels with QR codes to stick on containers, so that you always know where everything is.
- Add parts to project from a kicad BOM file.
- Always know how many of each project you can make with the parts in stock.

# Prepare your supabase instance
Create an account and a project on supabase.com or self-host an instance using [their documentation](https://supabase.com/docs/guides/self-hosting).

Run the tables.sql file contents in the SQL editor of your supabase instance.

# Run with Docker
A docker image is available on docker hub
Run it like this: `docker run -p 3000:3000 -e SUPABASE_URL=https://yourproject.supabase.co -e SUPABASE_KEY=a_very_long_key_here -e BASE_URL=http://localhost:3000 jonathanfoucher/inventronic:latest`

Or create a `.env` file and use this command:
`docker run -p 3000:3000 --env-file=.env jonathanfoucher/inventronic:latest`

# Use Inventronic

You can now login with OTP by going to http://localhost:3000/