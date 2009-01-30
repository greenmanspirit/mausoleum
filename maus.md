#!../minid/bin/mdcl
module maus

global packageInfo = {}
global mode = "";

function main(vararg)
{
	parse_arguments(vararg)
	parse_metadata();
	switch(mode)
	{ 
		case "install":
			os.system("git clone "~packageInfo["repo"])
			writefln("moving into {}",packageInfo["name"])
			io.changeDir(packageInfo["name"])
			os.system("git checkout "~packageInfo["commit-object"])
			if(!isNull(packageInfo["patch-repo"]))
				os.system("git pull "~packageInfo["patch-repo"])
			for(step: 0..#packageInfo["installSteps"])
				os.system(packageInfo["installSteps"][step])
			writefln("moving out of {}",packageInfo["name"])
			io.changeDir("..")
			break;
		case "uninstall":
			writefln("Running {} uninstallation",packageInfo["name"]);
			for(step: 0..#packageInfo["uninstallSteps"])
				os.system(packageInfo["uninstallSteps"][step])
			break;
		case "search":
			break;
		default:
			break;
	}
}

function parse_arguments(vararg)
{
	for(i: 0..#vararg)
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
			default:
				writefln("Invalid argument: {}",currentArg)
				break
		}
		if(mode != "")
			break
	}
}

function parse_metadata()
{
	local fileContents = io.readFile(package~".maus").strip().splitLines()
	
	local equalRegExp = regexp.Regexp("(.+)=(.+)", "g")
	local numInstallSteps = 0
	local numUninstallSteps = 0
	local gettingInstallLines = 0
	local gettingUninstallLines = 0

	for(i: 0..#fileContents)
	{
		local currentLine = fileContents[i];
		if(gettingInstallLines < numInstallSteps)
		{
			packageInfo["installSteps"][gettingInstallLines] = currentLine
			gettingInstallLines++
		}
		else if(gettingUninstallLines < numUninstallSteps)
		{
			packageInfo["uninstallSteps"][gettingUninstallLines] = currentLine
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
					packageInfo["installSteps"] = array.new(numInstallSteps)
					break
				case "uninstall-steps":
					numUninstallSteps = toInt(value)
					packageInfo["uninstallSteps"] = array.new(numUninstallSteps)
					break
				default:
					writefln("WTF? \"{}\"",name)
			}
		}
		else
		{
			writefln("Invalid metadata: {}",currentLine);
			return 1;
		}
	}
}
