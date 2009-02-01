#!/bin/mdcl
module maus

global packageInfo = {}
global mode = "";

function main(vararg)
{
	parse_arguments(vararg)
	switch(mode)
	{ 
		case "install":
			parse_metadata()
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
			parse_metadata()
			writefln("Running {} uninstallation",packageInfo["name"])
			for(step: 0..#packageInfo["uninstallSteps"])
				os.system(packageInfo["uninstallSteps"][step])
			break;
		case "search":
			io.changeDir("maus_packages");
			local packages = io.listFiles(".", "/mausoleum/maus_packages/*"~searchTerm~"*.maus")
			for(i: 0..#packages)
			{
					writefln("{}",packages[i][25..#packages[i]-5])
			}
			io.changeDir("..");
			break
		case "update":
			io.changeDir("maus_packages")
			os.system("git pull origin master")
			break
		default:
			break
	}
}

function parse_arguments(vararg)
{
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
			mode = ""
			break;
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
		local currentLine = fileContents[i]
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
					writefln("Unknown Metadata (ignoring): \"{}\"",name)
			}
		}
		else
		{
			writefln("Invalid metadata: {}",currentLine)
			return 1
		}
	}
}
