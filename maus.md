#!mdcl
module maus

global exit = thread.halt

global packageInfo = {}
global mode = ""

function main(vararg)
{
	parse_arguments(vararg)
	switch(mode)
	{ 
		case "install":
			parse_metadata()
			writefln("Running {} installation",packageInfo["name"])
			os.system("git clone "~packageInfo["repo"])
			writefln("moving into {}",packageInfo["name"])
			io.changeDir(packageInfo["name"])
			os.system("git checkout "~packageInfo["commit-object"])
			if(!isNull(packageInfo["patch-repo"]))
				os.system("git pull "~packageInfo["patch-repo"])
			loadString(packageInfo["installScript"])()
			io.changeDir("..")
			break
		case "uninstall":
			parse_metadata()
			writefln("Running {} uninstallation",packageInfo["name"])
			loadString(packageInfo["uninstallScript"])()
			break
		case "search":
			io.changeDir("packages")
			local dirs = io.listDirs(".")
			for(i: 0..#dirs)
			{
				io.changeDir(dirs[i])
				local packages = io.listFiles(".", dirs[i]~"/*"~searchTerm~"*.maus")
				for(a: 0..#packages)
				{
						writefln("{}",io.name(packages[a]))
				}
				io.changeDir("..")
			}
			io.changeDir("..")
			break
		case "update":
			local fileContents = io.readFile("repos.lst").strip().splitLines()
			io.changeDir("packages")
			for(i: 0..#fileContents)
			{
				local repoInfo = fileContents[i].split()
				local address = repoInfo[0]
				local name = repoInfo[1]

				writefln("{}",io.currentDir())

				if(io.exists(name) && io.isDir(name))
				{
					io.changeDir(name)
					os.system("git pull origin master")
					io.changeDir("..")
				}
				else
				{
					os.system("rm -rf "~name)
					os.system("git clone "~address)
				}
				
			}
			io.changeDir("..")
			break
		default:
			break
	}
}

function parse_arguments(vararg)
{
	if(#vararg == 0)
	{
		writefln("I need arguments to run, I'm pretty useless without them")
		exit();
	}
	for(i: 0..#vararg)
	{
		try
		{
			local currentArg = vararg[i];
			switch(currentArg)
			{
				case "--install":
				case "-i":
					mode = "install"
					global package = vararg[i+1]
					break
				case "--unistall":
				case "-u":
					mode = "uninstall"
					global package = vararg[i+1]
					break
				case "--search":
				case "-s":
					mode = "search"
					global searchTerm = vararg[i+1]
					break
				case "--update":
				case "-p":
					mode = "update"
					break
				default:
					writefln("Invalid argument: {}",currentArg)
					break
			}
		}
		catch(e)
		{
			writefln("Not enough arguments")
			exit()
			break
		}
		if(mode != "")
			break
	}
}

function find_package(package)
{
	io.changeDir("packages")
	local dirs = io.listDirs(".")
	for(i: 0..#dirs)
	{
		io.changeDir(dirs[i])
		local packages = io.listFiles(".", dirs[i]~"/"~package~".maus")
		if(#packages > 0)
			return packages[0]
		io.changeDir("..")
	}
	io.changeDir("..")
	return ""
}

function parse_metadata()
{
	try
	{
		global fileContents = io.readFile(find_package(package)).strip().splitLines()
	}
	catch(e)
	{
		writefln("Package does not exist")
		exit()
	}

	local equalRegExp = regexp.Regexp("(.+)=(.+)", "g")
	local numInstallSteps = 0
	local numUninstallSteps = 0
	local gettingInstallLines = 0
	local gettingUninstallLines = 0
	
	for(i: 0..#fileContents)
	{
		local currentLine = fileContents[i]
		if(currentLine.startsWith("#"))
		{
			continue
		}
		else if(gettingInstallLines < numInstallSteps)
		{
			packageInfo["installScript"] ~= currentLine~"\n"
			gettingInstallLines++
		}
		else if(gettingUninstallLines < numUninstallSteps)
		{
			packageInfo["uninstallScript"] ~= currentLine~"\n"
			gettingUninstallLines++
		}
		else if(equalRegExp.test(currentLine))
		{
			local name = equalRegExp.match(1).strip()
			local value = equalRegExp.match(2).strip()
			switch(name)
			{
				case "name":
					packageInfo["name"] = value
					break
				case "version":
					packageInfo["version"] = value
					break
				case "commit-object":
					packageInfo["commit-object"] = value
					break
				case "repo":
					packageInfo["repo"] = value
					break
				case "patch-repo":
					packageInfo["patch-repo"] = value
					break
				case "install-steps":
					numInstallSteps = toInt(value)
					packageInfo["installScript"] = ""
					break
				case "uninstall-steps":
					numUninstallSteps = toInt(value)
					packageInfo["uninstallScript"] = ""
					break
				case "patch-object":
					packageInfo["patch-object"] = value
					break
				case "test-command":
					break
				case "dependency":
					break
				case "min-rollback-object":
					break
				case "min-patch-rollback-object":
					break
				case "rollback-patch-equivalence":
					break
				default:
					writefln("Unknown Metadata (ignoring): \"{}\"",name)
			}
		}
		else
		{
			writefln("Invalid metadata: {}",currentLine)
		}
	}
}
