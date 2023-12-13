# Networks


```cs
using System;
using System.Collections.Generic;
using System.IO;
using System.Net;
using System.Net.Sockets;
using System.Threading;

namespace YourNamespace
{
    class Program
    {
        private static bool debug = false;
        private static Logger log = new Logger();

        private static Dictionary<string, User> DataBase = new Dictionary<string, User>
        {
            ["cssbct"] = new User
            {
                UserID = "cssbct",
                Surname = "Tompsett",
                Fornames = "Brian C",
                Title = "Eur Ing",
                Position = "Lecturer of Computer Science",
                Phone = "+44 1482 46 5222",
                Email = "B.C.Tompsett@hull.ac.uk",
                Location = "in RB-336"
            }
        };

        private static string connStr = "server=localhost;user=root;database=world;port=3306;password=L3tM31n";
        private static MySqlConnection conn = new MySqlConnection(connStr);

        static void Main(string[] args)
        {
            for (int i = 0; i < args.Length; i++)
            {
                Console.WriteLine(args[i]);
            }

            if (args.Length == 0)
            {
                Console.WriteLine("Starting Server");
                RunServer();
            }
            else
            {
                for (int i = 0; i < args.Length; i++)
                {
                    Console.WriteLine(args[i]);
                }
            }

            for (int i = 0; i < args.Length; i++)
            {
                ProcessCommand(args[i]);
            }
        }

        static void RunServer()
        {
            TcpListener listener = null;
            try
            {
                listener = new TcpListener(IPAddress.Any, 43);
                listener.Start();

                while (true)
                {
                    if (debug) Console.WriteLine("Server Waiting connection...");
                    TcpClient client = listener.AcceptTcpClient();

                    // Use a separate thread for each client
                    Thread clientThread = new Thread(() => HandleClient(client));
                    clientThread.Start();
                }
            }
            catch (Exception e)
            {
                log.Log(e.ToString());
            }
            finally
            {
                if (debug)
                    Console.WriteLine("Terminating Server");
                listener.Stop();
            }
        }

        static void HandleClient(TcpClient client)
        {
            try
            {
                NetworkStream socketStream = client.GetStream();
                doRequest(socketStream);
                socketStream.Close();
                client.Close();
            }
            catch (Exception e)
            {
                log.Log($"Error handling client: {e}");
            }
        }

        // Handle a network request
        static void doRequest(NetworkStream socketStream)
        {
            try
            {
                using (StreamWriter sw = new StreamWriter(socketStream))
                using (StreamReader sr = new StreamReader(socketStream))
                {
                    socketStream.ReadTimeout = 1000;

                    // Logic for network update function
                    string updateCommand = "update"; // Placeholder command
                    ProcessNetworkUpdate(updateCommand);

                    if (debug) Console.WriteLine("Waiting for input from client...");
                    String line = sr.ReadLine();
                    Console.WriteLine($"Received Network Command: '{line}'");

                    if (line == null)
                    {
                        if (debug) Console.WriteLine("Ignoring null command");
                        return;
                    }

                    // sw.WriteLine(line);   // Need to remove this line after testing
                    // sw.Flush();           // Need to remove this line after testing  
                    if (line == "POST / HTTP/1.1")
                    {
                        // The we have an update
                        if (debug) Console.WriteLine("Received an update request");
                    }
                    else if (line.StartsWith("GET /?name=") && line.EndsWith(" HTTP/1.1"))
                    {
                        // then we have a lookup
                        if (debug) Console.WriteLine("Received a lookup request");
                    }
                    else
                    {
                        // We have an error
                        Console.WriteLine($"Unrecognized command: '{line}'");
                    }

                    sw.WriteLine("HTTP/1.1 400 Bad Request");
                    sw.WriteLine("Content-Type: text/plain");
                    sw.WriteLine();

                    String[] slices = line.Split(" ");  // Split into 3 pieces
                    String ID = slices[1].Substring(7);  // start at the 7th letter of the middle slice - skip `/?name=`

                    if (DataBase.ContainsKey(ID))
                    {
                        String result = DataBase[ID].Location;
                        sw.WriteLine("HTTP/1.1 200 OK");
                        sw.WriteLine("Content-Type: text/plain");
                        sw.WriteLine();
                        sw.WriteLine(result);
                        sw.Flush();
                        Console.WriteLine($"Performed Lookup on '{ID}' returning '{result}'");
                    }
                    else
                    {
                        // Not found
                        sw.WriteLine("HTTP/1.1 404 Not Found");
                        sw.WriteLine("Content-Type: text/plain");
                        sw.WriteLine();
                        sw.Flush();
                        Console.WriteLine($"Performed Lookup on '{ID}' returning '404 Not Found'");
                    }
                    DataBase["cssbct"].Location = "network update test string";
                }
            }
            catch (Exception e)
            {
                log.Log($"Error processing network request: {e}");
            }
        }

        static void ProcessNetworkUpdate(string command)
        {
            // logic for network update function
            Console.WriteLine($"Processing network update: {command}");
        }

        static void ProcessCommand(string command)
        {
            if (debug) Console.WriteLine($"\nCommand: {command}");
            try
            {
                String[] slice = command.Split(new char[] { '?' }, 2);
                String ID = slice[0];
                String operation = null;
                String update = null;
                String field = null;

                if (slice.Length == 2)
                {
                    operation = slice[1];
                    String[] pieces = operation.Split(new char[] { '=' }, 2);
                    field = pieces[0];
                    if (pieces.Length == 2) update = pieces[1];
                }

                Console.Write($"Operation on ID '{ID}'");

                if (operation == null)
                {
                    Console.WriteLine(" output all fields");
                }
                else if (update == null)
                {
                    Console.WriteLine($" lookup field '{field}'");
                }
                else
                {
                    Console.WriteLine($" update field '{field}' to '{update}'");
                }

                Console.Write($"Operation on ID '{ID}'");

                if (operation == null) Dump(ID);
                else if (update == null) Lookup(ID, field);
                else Update(ID, field, update);
            }
            catch (Exception e)
            {
                Console.WriteLine($"Fault in Command Processing: {e.ToString()}");
            }
        }

        static void Dump(String ID)
        {
            Console.WriteLine(" output all fields");
            Console.WriteLine($"UserID={DataBase[ID].UserID}");
            Console.WriteLine($"Surname={DataBase[ID].Surname}");
            Console.WriteLine($"Fornames={DataBase[ID].Fornames}");
            Console.WriteLine($"Title={DataBase[ID].Title}");
            Console.WriteLine($"Position={DataBase[ID].Position}");
            Console.WriteLine($"Phone={DataBase[ID].Phone}");
            Console.WriteLine($"Email={DataBase[ID].Email}");
            Console.WriteLine($"location={DataBase[ID].Location}");
        }

        static void Lookup(String ID, String field)
        {
            if (debug)
                Console.WriteLine($" lookup field '{field}'");

            String result = null;
            switch (field)
            {
                case "location": result = DataBase[ID].Location; break;
                case "UserID": result = DataBase[ID].UserID; break;
                case "Surname": result = DataBase[ID].Surname; break;
                case "Fornames": result = DataBase[ID].Fornames; break;
                case "Title": result = DataBase[ID].Title; break;
                case "Phone": result = DataBase[ID].Phone; break;
                case "Position": result = DataBase[ID].Position; break;
                case "Email": result = DataBase[ID].Email; break;
            }
            Console.WriteLine(result);
        }

        static void Update(String ID, String field, String update)
        {
            if (debug)
                Console.WriteLine($" update field '{field}' to '{update}'");

            switch (field)
            {
                case "location": DataBase[ID].Location = update; break;
                case "UserID": DataBase[ID].UserID = update; break;
                case "Surname": DataBase[ID].Surname = update; break;
                case "Fornames": DataBase[ID].Fornames = update; break;
                case "Title": DataBase[ID].Title = update; break;
                case "Phone": DataBase[ID].Phone = update; break;
                case "Position": DataBase[ID].Position = update; break;
                case "Email": DataBase[ID].Email = update; break;
            }
            Console.WriteLine("OK");
        }
    }

    class User
    {
        public string UserID;
        public string Surname;
        public string Fornames;
        public string Title;
        public string Position;
        public string Phone;
        public string Email;
        public string Location;
    }

    class Logger
    {
        public void Log(string message)
        {
            Console.WriteLine($"Error: {message}");
        }
    }
}
```
