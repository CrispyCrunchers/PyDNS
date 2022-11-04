#!/usr/bin/python


## dns-control
##
## Written By: Chris Kloiber
## Date: 10/1/2019
##
## Description: 
## This script helps add/remove Bind DNS zones, 
## as well as add/remove zone file records


### USER VARS ###

named_config_file = "/etc/named.conf"
named_zone_definition_path = "/etc/named.zones.d"
named_zone_path = "/var/named/zones"
named_zone_definition_file_prefix = "zone"
named_zone_file_prefix = "db"

provider_ns1 = ""
provider_ns2 = ""
provider_postmaster = ""

named_master_ip = ""
named_slave_ip = ""
named_slave_auto_update_records = True
named_slave_ssh_username = "root"
named_slave_ssh_control_script = "/var/scripts/dns-control"

ssh_binary = "/usr/bin/ssh"

### END USER REGION ###


### PRE-DEFINED VARS ###

named_zone_definition_file = ""
named_zone_file = ""

### END PRE-DEFNIED VARS ###




### DO NOT EDIT AFTER THIS LINE (Unless you know what you are doing :)) ###
### BEGIN CODE ###

import os
import sys
import subprocess
import datetime
import re



### FUNCTION DEFINITIONS ###

def printHelp():
        print "Command Line Arguments"
        print "----------------------------"
        print "-is <domain.com>\t:\tIncrement $domain serial"
        print "-rb\t\t\t:\tReloads the DNS Server"
        print ""
        print "-ld\t\t\t:\tList domains"
        print "-nd <domain.com>\t:\tNew Domain: Starts the wizard to setup $domain"
        print "-rd <domain.com>\t:\tRemove Domain: Deletes $domain"
        print ""
        print "-lr <domain.com>\t:\tList Domain Records: Lists all the domains DNS records."
        print "-nr <domain.com>\t:\tNew Record: Adds a DNS record to a DNS domain."
        print "-dr <domain.com>\t:\tDelete record: Removed a DNS record from a DNS domain."
        print ""
        print "-ns <domain.com> <master_server_ip>\t:\tNew Slave Domain from Master Server"

def unknownArguments():
        print "Unknown command line argument"
        print "Use -h or --help for help"
        print ""
        printHelp()

def missingArguments():
        print "Missing the <domain>"
        print ""
        printHelp()

def listDomains():
        print ""
        print "Listing all domains: "
        print ""

        zone_files = os.listdir(named_zone_definition_path)
        for file in zone_files:
                print file

def newDomain(domain):
        print ""
        print "Wizard for " + domain
        print ""

        setup_initial_A_record = get_bool("Do you want to enter an initial A record? ")
        if setup_initial_A_record:
                initial_A_record = raw_input("Enter the IP for the initial A record (www): ")

        setup_initial_MX_record = get_bool("Do you want to enter an initial MX record? ")
        if setup_initial_MX_record:
                initial_MX_record = raw_input("Enter the FQDN of the mail server: ")

        print ""
        print "Creating DNS records for " + domain

        # Write include line for zone definition file in named_config_file
        writeLine(named_config_file,'include "' + named_zone_definition_file + '";')

        # Create zone definition file
        f = open(named_zone_definition_file,"w+")
        f.write('zone "' + domain + '"{\n')
        f.write('\ttype master;\n')
        f.write('\tfile "' + named_zone_file + '";\n')
        f.write('};\n')
        f.close()

        # Crate zone file

        d = datetime.date.today()
        serial = '%02d' % d.year + '%02d' % d.month + '%02d' % d.day + "01"


        f = open(named_zone_file,"w+")
        f.write('$ORIGIN ' + domain + '.\n')
        f.write('$TTL 86400\n')
        f.write('@\tIN\tSOA\t' + provider_ns1 + '.\t' + provider_postmaster + '. (\n')
        f.write('\t\t' + serial + '\t; serial\n')
        f.write('\t\t3600\t\t; refresh\n')
        f.write('\t\t7200\t\t; retry\n')
        f.write('\t\t604800\t\t; expire\n')
        f.write('\t\t86400   )\t; default_ttl\n')
        f.write('\n\n; NS Records\n')
        f.write('@\t\t\tNS\t' + provider_ns1 + '.\n')
        f.write('@\t\t\tNS\t' + provider_ns2 + '.\n')
        f.write('\n\n\n; CUSTOM ENTRIES\n; DO NOT EDIT THIS LINE\n')
        if setup_initial_MX_record: 
                f.write('@\t\t\tMX\t10\t' + initial_MX_record + '.\n')
        if setup_initial_A_record:
                f.write('@\t\t\tA\t' + initial_A_record + '\n')
                f.write('www\t\t\tA\t' + initial_A_record + '\n')

        f.close()

        reloadBind()

        if named_slave_auto_update_records == True:
                named_slave_server_command = named_slave_ssh_control_script + " -ns " + domain + " " + named_master_ip
                updateSlaveServer(named_slave_server_command)

        print ""
        print "Domain Added Successfully"

def updateSlaveServer(command):
        print ""
        print ""
        print "Updating slave name server (NS2)."
        print "Server: " + named_slave_ip
        print ""


        ssh = subprocess.Popen([ssh_binary, "%s" % named_slave_ip, command],
                shell=False,
                stdout=subprocess.PIPE,
                stderr=subprocess.PIPE)
        result = ssh.stdout.readlines()
        if result == []:
                error = ssh.stderr.readlines()
                print >>sys.stderr, "ERROR: %s" % error
        else:
                #print result
                print ""
                print ""
                print "Successfully updated slave server"

def newSlaveDomain(domain,master_server_ip):
        print ""
        print "Adding slave domain..."

        # Write include line for zone definition file in named_config_file
        writeLine(named_config_file,'include "' + named_zone_definition_file + '";')

        f = open(named_zone_definition_file,"w+")
        f.write('zone "' + domain + '"{\n')
        f.write('\ttype slave;\n')
        f.write('\tfile "' + named_zone_file + '";\n')
        f.write('\tmasters{\n')
        f.write('\t\t' + master_server_ip + ';\n')
        f.write('\t};\n')
        f.write('};\n')
        f.close()

        print "Slave added successfully"

        reloadBind()

def removeDomain(domain,force=False):
        print "Removing DNS config for domain: " + domain

        if force == False:
                delete_sure = get_bool("Are you sure you want to delete the DNS records? [y/N]: ")
        else:
                delete_sure = True

        if delete_sure == True:
                # Delete include line from named_config_file (eg.: /etc/named.conf)
                deleteLineByPhrase(named_config_file,'include "' + named_zone_definition_file + '";\n')

                # Delete domain zone definition file (eg.: /etc/named.zones/zone.domain)
                os.remove(named_zone_definition_file)

                # Delete domain zone file (eg.: /var/named/zones/db.domain)
                os.remove(named_zone_file)

                # Reload BIND
                reloadBind()

                # Update slave server, if required
                if named_slave_auto_update_records == True:
                        named_slave_server_command = named_slave_ssh_control_script + " -rds " + domain
                        updateSlaveServer(named_slave_server_command)

                print ""
                print "Removal complete"

        else:
                print ""
                print "Domain NOT deleted."


def listDNSRecords(domain):

        print "Listing DNS records for domain: " + domain
        print ""
        print ""

        beginning_line_number = getLineNumberByPhrase(named_zone_file,"; DO NOT EDIT THIS LINE")
        beginning_line_number += 1

        offset_line_number = 1
        with open(named_zone_file) as f:
            for i in xrange(beginning_line_number):
                f.next()
            for line in f:
                print str(offset_line_number) + ") " + line
                offset_line_number += 1

def newDNSRecord(domain):
        print ""
        print "Adding new record for: " + domain
        print ""

        record_type_valid = False
        while record_type_valid == False:
                record_type = raw_input("Enter record type: (A, CNAME, MX): ").upper()

                if record_type == "A":
                        record_type_valid = True
                        A_record = raw_input("A Record: (@=domain.com, www=www.domain.com): ").lower()
                        A_record_IP = raw_input("Record IP: ")

                        addARecord(domain,A_record,A_record_IP)

                elif record_type == "CNAME":
                        record_type_valid = True
                        CNAME_record = raw_input("CNAME Record: (@=domain.com, www=www.domain.com): ").lower()
                        CNAME_record_FQDN = raw_input("Record FQDN: ").lower()

                        addCNAMERecord(domain,CNAME_record,CNAME_record_FQDN)

                elif record_type == "MX":
                        record_type_valid = True
                        MX_record = raw_input("MX record (@=domain.com, mail=mail.domain.com): ").lower()
                        MX_priority = raw_input("MX priority: ")
                        MX_server = raw_input("MX Server FQDN: ").lower()

                        addMXRecord(domain,MX_record,MX_priority,MX_server)
                else:
                        print "Invalid record type, please re-enter."

        incrementDNSRecordSerial(domain)

def addARecord(domain,record,IP):
        writeLine(named_zone_file, record + '\t\t\tA\t' + IP + '\r')

def addCNAMERecord(domain,record,FQDN):
        writeLine(named_zone_file, record + '\t\t\tCNAME\t' + FQDN + '.\r')

        #print "Record Added."                
        #print "Domain: " + domain + " Type: " + record_type + "Record: " + A_record + " IP: " + A_record_IP                
        #print ""                

def addMXRecord(domain,record,priority,FQDN):
        writeLine(named_zone_file, record + '\t\t\tMX\t' + priority + '\t' + FQDN + '.\r')

def deleteDNSRecord(domain):
        listDNSRecords(domain)
        beginning_line_number = getLineNumberByPhrase(named_zone_file,"; DO NOT EDIT THIS LINE")

        print ""
        deleteLineNumber = raw_input("Choose a entry number to remove: ")

        print ""
        print "Removing entry number: " + str(deleteLineNumber)

        deleteLineNumber = int(deleteLineNumber)
        deleteLineNumber += beginning_line_number
        deleteLineByLineNumber(named_zone_file,deleteLineNumber)

        print "Removed"

        incrementDNSRecordSerial(domain)

def incrementDNSRecordSerial(domain):
        print ""
        print "Incrementing Serial for: " + domain
        print ""

        lineNumber = getLineNumberByPhrase(named_zone_file,"; serial")
        #print "Line Number: " + str(lineNumber)

        originalLine = getLineByLineNumber(named_zone_file, lineNumber)
        #print "Original Line: " + originalLine

        originalLineArr = originalLine.split(";")
        originalSerial = originalLineArr[0]

        originalSerial = re.sub("[^0-9]", "", originalSerial)
        originalSerial = int(originalSerial)
        print "Original Serial: " + str(originalSerial)

        newSerial = originalSerial + 1
        print "New Serial: " + str(newSerial)
        updateLineByLineNumber(named_zone_file,lineNumber,'\t\t' + str(newSerial) + '\t; serial\n')

        print ""
        print "Serial Number Updated"

        reloadBind()

def reloadBind():
        print ""
        print "Reloading named..."

        subprocess.call(["systemctl","restart","named"])
        print "Complete"

def get_record_type():
        return input("Enter record type: (A, CNAME, MX): ".upper())

def get_bool(prompt):
    while True:
        try:
           return {"y":True,"n":False,"":False}[raw_input(prompt).lower()]
        except KeyError:
           print "Invalid input please enter True or False!"

def get_int(prompt):
    while True:
        try:
           return int(input(prompt))
        except ValueError:
           print "Please enter an integer value"

def getLineByLineNumber(file,lineNumber):
    with open(file,'r') as f:
        for (i, line) in enumerate(f):
            if i == lineNumber:
                return line
    return -1


def getLineNumberByPhrase(file,phrase):
    with open(file,'r') as f:
        for (i, line) in enumerate(f):
            if phrase in line:
                return i
    return -1

def updateLineByLineNumber(file,lineNumber,newLine):
        with open(file,'r+') as f:
                content = f.readlines()
                f.seek(0)
                for (i, line) in enumerate(content):
                        if i == int(lineNumber):
                                f.write(newLine)
                        else:
                                f.write(line)
                f.truncate()
                f.close()

def deleteLineByPhrase(file,deleteLinePhrase):
        with open(file,'r+') as f:
                content = f.readlines()
                f.seek(0)
                for (i, line) in enumerate(content):
                        if line != deleteLinePhrase:
                                f.write(line)
                f.truncate()
                f.close()

def deleteLineByLineNumber(file,deleteLineNumber):
        with open(file,'r+') as f:
                content = f.readlines()
                f.seek(0)
                for (i, line) in enumerate(content):
                        if i != int(deleteLineNumber):
                                f.write(line)
                f.truncate()
                f.close()

def writeLine(file,writeLine):
        with open(file, 'a+') as f:
                f.write(writeLine + '\n')
                f.close()

### END FUNCTION DEFS ###



### MAIN PROGRAM ###


print ""
print "DNS Control v1"
print "-----------------------------------------------------"
print ""


### Handle arguments ###

if len(sys.argv) < 2:
        printHelp()

elif len(sys.argv) == 2: 
        # Check for help args
        if sys.argv[1] == "-h" or sys.argv[1] == "--help":
                printHelp()

        ## List Domains
        elif sys.argv[1] == "-ld":
                listDomains()

        elif sys.argv[1] == "-rb":
                reloadBind()

        else:
                unknownArguments()

elif len(sys.argv) == 3:
        domain = sys.argv[2]
        named_zone_definition_file = named_zone_definition_path + "/" + named_zone_definition_file_prefix + "." + domain
        named_zone_file = named_zone_path + "/" + named_zone_file_prefix + "." + domain

        ## New Domain
        if sys.argv[1] == "-nd":
                newDomain(domain)

        ## Remove domain
        elif sys.argv[1] == "-rd":
                removeDomain(domain)

        ## Remove domain (forced -- from master server)
        elif sys.argv[1] == "-rds":
                removeDomain(domain,force=True)

        ## List Domain Records
        elif sys.argv[1] == "-lr":
                listDNSRecords(domain)

        ## New domain records
        elif sys.argv[1] == "-nr":
                newDNSRecord(domain)

        ## Delete domain records
        elif sys.argv[1] == "-dr":
                deleteDNSRecord(domain)

        ## Increment Serial Number
        elif sys.argv[1] == "-is":
                incrementDNSRecordSerial(domain)

        else:
                unknownArguments()

elif len(sys.argv) == 4:
        domain = sys.argv[2]
        named_zone_definition_file = named_zone_definition_path + "/" + named_zone_definition_file_prefix + "." + domain
        named_zone_file = named_zone_path + "/" + named_zone_file_prefix + "." + domain

        if sys.argv[1] == "-ns":
                named_master_server = sys.argv[3]

                newSlaveDomain(domain,named_master_server)
        else:
                unknownArguments()


## Catch case / invalid args
else:
        unknownArguments()

print ""
sys.exit()

### END main() ###
