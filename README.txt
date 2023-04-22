Steps :
Install external credential storage plugin on your instance if its PDI you can do it from developer.servicenow.com if its client instance
you can request from support.servicenow.com
Create JAR file that can resolve credentials
    Perform following steps on Midserver
        1. Install Java SE 8 on your midserver VM.
        2. Set PATH For java bin folder
    Perform CredentialResolver setup
        1. Create environment variable on midserer VM.
            name = CREDENTIAL_RESOLVER_FILE value = <folder_name/dummycredentials.properties> // make sure to use / (forward slash) vs \
        2. Create a .java file with name = CredentialResolver.java with content mentioned below
        3. Create .class file by running following command by navigating to folder (C:/ServiceNow/API/Java/ or which ever folder has CredentialResolver.java)
            "C:\Program Files\Java\jdk-1.8\bin\javac.exe" -Xlint CredentialResolver.java (if javac.exe is installed in a different location use tht location)
        4. Create following folders C:/ServiceNow/API/Java/
            1. com
            2. under com -> snc
            3. under snc -> discovery
        5. Copy CredentialResolver.class file to C:/ServiceNow/API/Java/com/snc/discovery
        6. In folder C:/ServiceNow/API/Java/ run following command to create jar file
            C:\ServiceNow\API\Java>"C:\Program Files\Java\jdk-1.8\bin\jar.exe" cvf CredRes.jar ".\com\snc\discovery\CredentialResolver.class" 
        7. CredRes.jar file gets created
        8. Create a file with on vm with name = dummycredentials.properties (make sure it has .properties as extension)
        9. Content of above file should look like this
            disco_user.windows.pswd=?WDV;@6MjVbXytM2WiOu(dEFqZ-hhwLD
            disco_user.windows.user=.\\Administrator
            double \\ is used above to escape \ 


ServiceNow setup
    1. Create Credential Valut entry in table = vault_configuration
        name = CredRes 
                    This name should match name of jar file created in above steps
    2. Attach jar file to midserver jar Files
        table name = ecc_agent_jar
            name = CredRes
                This name should be same as jar file name
            version = 0.1
            Active = true
            Source = filebased  (optional field)
            Description = filebased credential resolver jar file (optional field)
    It takes about 5 to 10 min for midserver to load the jar to midserver VM
        Jar should be available in C:\ServiceNow\ServiceNow MID Server varanmidaws\agent\extlib (\agent\extlib) folder within midserver installed folder
    Once jar is available in above folder
    Create credentials in credential table 
        table name = windows_credentials
        name = disco user - external cred
        credential id = disco_user
        External credential store = true (check the box)
        Credential storage valut -> dropdown select CredRes
        Midserver -> select the midserver on which jar file is available (ideally jar file gets synced to all mids)
            pick a mid tat has access to the target windows VM for which credentials are provided in file = dummycredentials.credential
    Test the credetials, it should work
        





// Content of CredentialResolver.java
package com.snc.discovery;

import java.util.*;
import java.io.*;


/**
 * Basic implementation of a CredentialResolver that uses a properties file.
 */

public class CredentialResolver {

	private static String ENV_VAR = "CREDENTIAL_RESOLVER_FILE";
	//private static String DEFAULT_PROP_FILE_PATH = "C:\\dummycredentials.properties";
    private static String DEFAULT_PROP_FILE_PATH = "C:/ServiceNow/API/Java/dummycredentials.properties";

	// These are the permissible names of arguments passed INTO the resolve()
	// method.

	// the string identifier as configured on the ServiceNow instance...
	public static final String ARG_ID = "id";

	// a dotted-form string IPv4 address (like "10.22.231.12") of the target
	// system...
	public static final String ARG_IP = "ip";

	// the string type (ssh, snmp, etc.) of credential as configured on the
	// instance...
	public static final String ARG_TYPE = "type";

	// the string MID server making the request, as configured on the
	// instance...
	public static final String ARG_MID = "mid";

	// These are the permissible names of values returned FROM the resolve()
	// method.

	// the string user name for the credential, if needed...
	public static final String VAL_USER = "user";

	// the string password for the credential, if needed...
	public static final String VAL_PSWD = "pswd";

	// the string pass phrase for the credential if needed:
	public static final String VAL_PASSPHRASE = "passphrase";

	// the string private key for the credential, if needed...
	public static final String VAL_PKEY = "pkey";

	// the string authentication protocol for the credential, if needed...
	public static final String VAL_AUTHPROTO = "authprotocol";

	// the string authentication key for the credential, if needed...
	public static final String VAL_AUTHKEY = "authkey";

	// the string privacy protocol for the credential, if needed...
	public static final String VAL_PRIVPROTO = "privprotocol";

	// the string privacy key for the credential, if needed...
	public static final String VAL_PRIVKEY = "privkey";


	private Properties fProps;

	public CredentialResolver() {
	}

	private void loadProps() {
		if(fProps == null)
			fProps = new Properties();

		try {
			String propFilePath = System.getenv(ENV_VAR);
			if(propFilePath == null) {
				System.err.println("Environment var "+ENV_VAR+" not found. Using default file: "+DEFAULT_PROP_FILE_PATH);
				propFilePath = DEFAULT_PROP_FILE_PATH;
			}

			File propFile = new File(propFilePath);
			if(!propFile.exists() || !propFile.canRead()) {
				System.err.println("Can't open "+propFile.getAbsolutePath());
			}
			else {
				InputStream propsIn = new FileInputStream(propFile);
				fProps.load(propsIn);
			}
			//fProps.load(CredentialResolver.class.getClassLoader().getResourceAsStream("dummycredentials.properties"));
		} catch (IOException e) {
			System.err.println("Problem loading credentials file:");
			e.printStackTrace();
		}
	}

	/**
	 * Resolve a credential.
	 */
	public Map resolve(Map args) {
		loadProps();
		String id = (String) args.get(ARG_ID);
		String type = (String) args.get(ARG_TYPE);
		String keyPrefix = id+"."+type+".";

		if(id.equalsIgnoreCase("misbehave"))
			throw new RuntimeException("I've been a baaaaaaaaad CredentialResolver!");

		// the resolved credential is returned in a HashMap...
		Map result = new HashMap();
		result.put(VAL_USER, fProps.get(keyPrefix + VAL_USER));
		result.put(VAL_PSWD, fProps.get(keyPrefix + VAL_PSWD));
		result.put(VAL_PKEY, fProps.get(keyPrefix + VAL_PKEY));
		result.put(VAL_PASSPHRASE, fProps.get(keyPrefix + VAL_PASSPHRASE));
		result.put(VAL_AUTHPROTO, fProps.get(keyPrefix + VAL_AUTHPROTO));
		result.put(VAL_AUTHKEY, fProps.get(keyPrefix + VAL_AUTHKEY));
		result.put(VAL_PRIVPROTO, fProps.get(keyPrefix + VAL_PRIVPROTO));
		result.put(VAL_PRIVKEY, fProps.get(keyPrefix + VAL_PRIVKEY));

		System.err.println("Error while resolving credential id/type["+id+"/"+type+"]");

		return result;
	}


	/**
	 * Return the API version supported by this class.
	 */
	public String getVersion() {
		return "1.0";
	}

	public static void main(String[] args) {
		CredentialResolver obj = new CredentialResolver();
		obj.loadProps();

		System.err.println("I spy the following credentials: ");
		for(Object key: obj.fProps.keySet()) {
			System.err.println(key+": "+obj.fProps.get(key));
		}

	}
}

To build the project use goal = package jar:jar
On Midserver 
1. Create environment variable with name = CREDENTIAL_RESOLVER_FILE value = D:\Platforms\ServiceNow (which ever is default location to fetch the file)
2. Deploy jar to midserver by uploading it to midserver jars on ServiceNow instance.

On ServiceNow instance
Install plugin : com.snc.discovery.external_credentials
Create a credential value choice option -> CredRes
Create a credential record type = Windows

Create a properties file with below content

disco_user.windows.pswd=?WDV;@6MjVbXytM2WiOu(dEFqZ-hhwLD
disco_user.windows.user=.\\Administrator

C:\ServiceNow\API\Java>"C:\Program Files\Java\jdk-1.8\bin\jar.exe" cvf CredRes.jar ".\com\snc\discovery\CredentialResolver.class"
added manifest
adding: com/snc/discovery/CredentialResolver.class(in = 4071) (out= 2220)(deflated 45%)
Error : 
java.lang.UnsupportedClassVersionError: com/snc/discovery/CredentialResolver has been compiled by a more recent version of the Java Runtime (class file version 64.0), 
this version of the Java Runtime only recognizes class file versions up to 55.0 at java.base/java.lang.ClassLoader.defineClass1(Native Method) at java.base/java.lang.ClassLoader.defineClass(ClassLoader.java:1017) at java.base/java.security.Secure

"C:\Program Files\Java\jdk-1.8\bin\jar.exe" tf .\CredRes.jar


References :
https://support.servicenow.com/kb?id=kb_article_view&sysparm_article=KB0714680
https://docs.servicenow.com/csh?topicname=external_cred_storage_configuration.html&version=latest
https://www.baeldung.com/maven-goals-phases
https://stackoverflow.com/questions/13616586/error-jar-is-not-recognized-as-an-internal-or-external-command-operable-progr
https://stackoverflow.com/questions/47457105/class-has-been-compiled-by-a-more-recent-version-of-the-java-environment
