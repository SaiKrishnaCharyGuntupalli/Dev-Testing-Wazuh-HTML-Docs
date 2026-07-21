# WAZUH MULTI-NODE RUNBOOK

## Overview
This runbook provides a step-by-step deployment and configuration guide for implementing a production-ready Wazuh multi-node architecture using Docker containers in Microsoft Azure. 

The deployment is designed to provide high availability, scalability, secure communication, centralized security monitoring, and efficient log storage management across multiple Wazuh components. 

The architecture consists of multiple Wazuh Indexer nodes distributed across hot and warm storage tiers, a clustered Wazuh Manager deployment with one master node and multiple worker nodes, a Wazuh Dashboard for centralized monitoring and management, and supporting components for secure log ingestion and traffic distribution. 

All communication between Wazuh components is secured using TLS certificates generated from a common Certificate Authority (CA). Dedicated certificates are created and distributed to the Wazuh Indexer nodes, Wazuh Manager nodes, Filebeat, and Wazuh Dashboard to provide encrypted and authenticated communication between services. 

The Wazuh Indexer cluster is configured with dedicated hot and warm nodes to support index lifecycle management and efficient storage utilization. Newly created security indices are initially stored on hot nodes for active ingestion and search operations and are later migrated to warm nodes according to configured Index State Management (ISM) policies. 

The deployment also integrates Azure Blob Storage with the Wazuh Indexer cluster using the Azure repository plugin. This integration enables index snapshots to be stored externally for backup, long-term retention, and disaster recovery purposes. 

The Wazuh Manager cluster provides centralized event analysis, agent management, rule processing, and security alert generation. The clustered architecture improves scalability and availability by distributing workloads between the master and worker nodes. 

The Wazuh Dashboard provides a centralized web interface for security monitoring, alert investigation, index management, and administration of the Wazuh environment. 

This runbook covers certificate generation and distribution, Wazuh Indexer cluster deployment, Wazuh Manager cluster deployment, Wazuh Dashboard deployment, Docker configuration, TLS configuration, Azure Blob Storage integration, index lifecycle management, snapshot configuration, and validation of the deployed environment. 

The objective of this document is to provide a repeatable deployment procedure that can be used by DevOps, SOC, and infrastructure teams to deploy, operate, troubleshoot, and maintain a secure and scalable Wazuh multi-node environment.

## Architecture diagram
![Multi node](<../../../assets/images/POC's/Vineeth\Wazuh Multi Node Runbook\Multi node.png>)


## 1. certificate generation guide
This guide helps you to understand the overview of certificate creation and deployment.

### 1.1 Creating  script file and generating the certs
Create a file as wazuh-certs-tool.sh and paste the below content
```
vi wazuh-certs-tool.sh
```
```
#!/bin/bash

# Wazuh installer
# Copyright (C) 2015, Wazuh Inc.
#
# This program is a free software; you can redistribute it
# and/or modify it under the terms of the GNU General Public
# License (version 2) as published by the FSF - Free Software
# Foundation.
adminpem="/etc/wazuh-indexer/certs/admin.pem"
adminkey="/etc/wazuh-indexer/certs/admin-key.pem"
readonly base_path="$(dirname "$(readlink -f "$0")")"
readonly config_file="${base_path}/config.yml"
readonly logfile="${base_path}/wazuh-certificates-tool.log"
cert_tmp_path="/tmp/wazuh-certificates"
debug=">> ${logfile} 2>&1"
readonly cert_tool_script_name=".*certs.*\.sh"

# ------------ certFunctions.sh ------------ 
function cert_validatePath() {
    local path="$1"
    local path_type="${2:-file}"

    # Check if path is empty
    if [[ -z "${path}" ]]; then
        common_logger -e "Path cannot be empty."
        return 1
    fi

    # Prevent path traversal attacks - reject paths with suspicious patterns
    if [[ "${path}" =~ \.\./|\.\.\\ ]]; then
        common_logger -e "Path traversal detected in: ${path}"
        return 1
    fi

    # Reject paths with newlines, carriage returns, or tabs (specific problematic characters)
    if [[ "${path}" =~ $'\n'|$'\r'|$'\t' ]]; then
        common_logger -e "Invalid characters detected in path: ${path}"
        return 1
    fi

    # For absolute paths validation
    if [[ "${path}" == /* ]]; then
        # Resolve to canonical path to prevent symlink attacks
        if command -v realpath >/dev/null 2>&1; then
            local canonical_path
            canonical_path=$(realpath -m "${path}" 2>/dev/null) || return 1

            # Ensure the canonical path doesn't escape expected boundaries
            if [[ ! "${canonical_path}" =~ ^/[a-zA-Z0-9/_.\-]+$ ]]; then
                common_logger -e "Invalid canonical path: ${canonical_path}"
                return 1
            fi
        fi
    fi

    return 0
}
function cert_sanitizeFilename() {
    local filename="$1"

    # Remove any path components
    filename="${filename##*/}"

    # Only allow alphanumeric, dash, underscore, and dot
    filename=$(echo "${filename}" | sed 's/[^a-zA-Z0-9._-]/_/g')

    # Prevent hidden files
    filename="${filename#.}"

    # Limit length to 255 characters
    if [[ ${#filename} -gt 255 ]]; then
        filename="${filename:0:255}"
    fi

    echo "${filename}"
}
function cert_sanitizeNodeName() {
    local nodename="$1"

    # Only allow alphanumeric, dash, underscore, and dot (typical for hostnames)
    if [[ ! "${nodename}" =~ ^[a-zA-Z0-9._-]+$ ]]; then
        common_logger -e "Invalid node name: ${nodename}. Only alphanumeric characters, dots, dashes, and underscores are allowed."
        return 1
    fi

    # Prevent names starting with dash or dot
    if [[ "${nodename}" =~ ^[-\.] ]]; then
        common_logger -e "Node name cannot start with dash or dot: ${nodename}"
        return 1
    fi

    # Limit length
    if [[ ${#nodename} -gt 253 ]]; then
        common_logger -e "Node name too long: ${nodename}"
        return 1
    fi

    return 0
}
function cert_cleanFiles() {

    common_logger -d "Cleaning certificate files."

    # Validate cert_tmp_path before use
    if ! cert_validatePath "${cert_tmp_path}" "directory"; then
        common_logger -e "Invalid certificate temporary path."
        exit 1
    fi

    # Remove files
    rm -f "${cert_tmp_path}"/*.csr
    rm -f "${cert_tmp_path}"/*.srl
    rm -f "${cert_tmp_path}"/*.conf
    rm -f "${cert_tmp_path}"/admin-key-temp.pem

}
function cert_checkOpenSSL() {

    common_logger -d "Checking if OpenSSL is installed."

    if [ -z "$(command -v openssl)" ]; then
        common_logger -e "OpenSSL not installed."
        exit 1
    fi

}
function cert_checkRootCA() {

    common_logger -d "Checking if the root CA exists."

    if  [[ -n ${rootca} || -n ${rootcakey} ]]; then
        # Verify variables match keys
        if [[ ${rootca} == *".key" ]]; then
            ca_temp=${rootca}
            rootca=${rootcakey}
            rootcakey=${ca_temp}
        fi

        # Validate paths
        if ! cert_validatePath "${rootca}" "file"; then
            common_logger -e "Invalid root CA certificate path: ${rootca}"
            cert_cleanFiles
            exit 1
        fi

        if ! cert_validatePath "${rootcakey}" "file"; then
            common_logger -e "Invalid root CA key path: ${rootcakey}"
            cert_cleanFiles
            exit 1
        fi

        if ! cert_validatePath "${cert_tmp_path}" "directory"; then
            common_logger -e "Invalid certificate temporary path."
            cert_cleanFiles
            exit 1
        fi

        # Validate that files exist
        if [[ -e ${rootca} ]]; then
            cp "${rootca}" "${cert_tmp_path}/root-ca.pem"
        else
            common_logger -e "The file ${rootca} does not exists"
            cert_cleanFiles
            exit 1
        fi
        if [[ -e ${rootcakey} ]]; then
            cp "${rootcakey}" "${cert_tmp_path}/root-ca.key"
        else
            common_logger -e "The file ${rootcakey} does not exists"
            cert_cleanFiles
            exit 1
        fi
    else
        cert_generateRootCAcertificate
    fi

}
function cert_executeAndValidate() {

    command_output=$("$@" 2>&1)
    e_code="${PIPESTATUS[0]}"

    if [ "${e_code}" -ne 0 ]; then
        common_logger -e "Error generating the certificates."
        common_logger -d "Error executing command: $@"
        common_logger -d "Error output: ${command_output}"
        cert_cleanFiles
        exit 1
    fi

}
function cert_generateAdmincertificate() {

    common_logger "Generating Admin certificates."

    # Validate cert_tmp_path
    if ! cert_validatePath "${cert_tmp_path}" "directory"; then
        common_logger -e "Invalid certificate temporary path."
        exit 1
    fi

    common_logger -d "Generating Admin private key."
    cert_executeAndValidate openssl genrsa -out "${cert_tmp_path}/admin-key-temp.pem" 4096
    common_logger -d "Converting Admin private key to PKCS8 format."
    cert_executeAndValidate openssl pkcs8 -inform PEM -outform PEM -in "${cert_tmp_path}/admin-key-temp.pem" -topk8 -nocrypt -v1 PBE-SHA1-3DES -out "${cert_tmp_path}/admin-key.pem"
    common_logger -d "Generating Admin CSR."
    cert_executeAndValidate openssl req -new -key "${cert_tmp_path}/admin-key.pem" -out "${cert_tmp_path}/admin.csr" -batch -subj '/C=US/L=Scottsdale/O=CISO/OU=SOC/CN=admin'
    common_logger -d "Creating Admin certificate."
    cert_executeAndValidate openssl x509 -days 1825 -req -in "${cert_tmp_path}/admin.csr" -CA "${cert_tmp_path}/root-ca.pem" -CAkey "${cert_tmp_path}/root-ca.key" -CAcreateserial -sha256 -out "${cert_tmp_path}/admin.pem"

}
function cert_generateCertificateconfiguration() {

    common_logger -d "Generating certificate configuration."

    local node_name="$1"

    # Validate node name
    if ! cert_sanitizeNodeName "${node_name}"; then
        common_logger -e "Invalid node name: ${node_name}"
        exit 1
    fi

    # Validate cert_tmp_path
    if ! cert_validatePath "${cert_tmp_path}" "directory"; then
        common_logger -e "Invalid certificate temporary path."
        exit 1
    fi

    cat > "${cert_tmp_path}/${node_name}.conf" <<- EOF
        [ req ]
        prompt = no
        default_bits = 4096
        default_md = sha256
        distinguished_name = req_distinguished_name
        x509_extensions = v3_req

        [req_distinguished_name]
        C = US
        L = Scottsdale
        O = CISO
        OU = SOC
        CN = cname

        [ v3_req ]
        authorityKeyIdentifier=keyid,issuer
        basicConstraints = CA:FALSE
        keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
        subjectAltName = @alt_names

        [alt_names]
        IP.1 = cip
	EOF


    conf="$(awk '{sub("CN = cname", "CN = '"${node_name}"'")}1' "${cert_tmp_path}/${node_name}.conf")"
    echo "${conf}" > "${cert_tmp_path}/${node_name}.conf"

    if [ "${#@}" -gt 1 ]; then
    sed -i '/IP.1/d' "${cert_tmp_path}/${node_name}.conf"

    san_index=1

    for (( i=2; i<=${#@}; i++ )); do

        isIP=$(echo "${!i}" | grep -P "^[0-9]{1,3}(\.[0-9]{1,3}){3}$")
        isDNS=$(echo "${!i}" | grep -P "^(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9-]*[a-zA-Z0-9])\.)*([A-Za-z0-9]|[A-Za-z0-9][a-zA-Z0-9-]*[A-Za-z0-9])\.([A-Za-z]{2,})$")

        if [ -n "${isIP}" ]; then
            printf "        IP.%d = %s\n" "${san_index}" "${!i}" >> "${cert_tmp_path}/${node_name}.conf"
            san_index=$((san_index+1))

        elif [ -n "${isDNS}" ]; then
            printf "        DNS.%d = %s\n" "${san_index}" "${!i}" >> "${cert_tmp_path}/${node_name}.conf"
            san_index=$((san_index+1))

        else
            common_logger -e "Invalid IP or DNS ${!i}"
            exit 1
        fi
    done

    #
    # Automatically add the node name as a DNS SAN
    #
    printf "        DNS.%d = %s\n" "${san_index}" "${node_name}" >> "${cert_tmp_path}/${node_name}.conf"

else
    common_logger -e "No IP or DNS specified"
    exit 1
fi

}
function cert_generateIndexercertificates() {

    if [ ${#indexer_node_names[@]} -gt 0 ]; then
        common_logger "Generating Wazuh indexer certificates."

        for i in "${!indexer_node_names[@]}"; do
            indexer_node_name=${indexer_node_names[$i]}

            # Validate node name
            if ! cert_sanitizeNodeName "${indexer_node_name}"; then
                common_logger -e "Invalid indexer node name: ${indexer_node_name}"
                exit 1
            fi

            common_logger -d "Creating the certificates for ${indexer_node_name} indexer node."
            cert_generateCertificateconfiguration "${indexer_node_name}" "${indexer_node_ips[i]}"
            common_logger -d "Creating the Wazuh indexer tmp key pair."
            cert_executeAndValidate openssl req -new -nodes -newkey rsa:4096 -keyout "${cert_tmp_path}/${indexer_node_name}-key.pem" -out "${cert_tmp_path}/${indexer_node_name}.csr" -config "${cert_tmp_path}/${indexer_node_name}.conf"
            common_logger -d "Creating the Wazuh indexer certificates."
            cert_executeAndValidate openssl x509 -req -in "${cert_tmp_path}/${indexer_node_name}.csr" -CA "${cert_tmp_path}/root-ca.pem" -CAkey "${cert_tmp_path}/root-ca.key" -CAcreateserial -out "${cert_tmp_path}/${indexer_node_name}.pem" -extfile "${cert_tmp_path}/${indexer_node_name}.conf" -extensions v3_req -days 1825
        done
    else
        return 1
    fi

}
function cert_generateFilebeatcertificates() {

    if [ ${#server_node_names[@]} -gt 0 ]; then
        common_logger "Generating Filebeat certificates."

        for i in "${!server_node_names[@]}"; do
            server_name="${server_node_names[i]}"

            # Validate node name
            if ! cert_sanitizeNodeName "${server_name}"; then
                common_logger -e "Invalid server node name: ${server_name}"
                exit 1
            fi

            common_logger -d "Generating the certificates for ${server_name} server node."
            j=$((i+1))
            declare -a server_ips=(server_node_ip_"$j"[@])
            cert_generateCertificateconfiguration "${server_name}" "${!server_ips}"
            common_logger -d "Creating the Wazuh server tmp key pair."
            cert_executeAndValidate openssl req -new -nodes -newkey rsa:4096 -keyout "${cert_tmp_path}/${server_name}-key.pem" -out "${cert_tmp_path}/${server_name}.csr" -config "${cert_tmp_path}/${server_name}.conf"
            common_logger -d "Creating the Wazuh server certificates."
            cert_executeAndValidate openssl x509 -req -in "${cert_tmp_path}/${server_name}.csr" -CA "${cert_tmp_path}/root-ca.pem" -CAkey "${cert_tmp_path}/root-ca.key" -CAcreateserial -out "${cert_tmp_path}/${server_name}.pem" -extfile "${cert_tmp_path}/${server_name}.conf" -extensions v3_req -days 1825
        done
    else
        return 1
    fi

}
function cert_generateDashboardcertificates() {
    if [ ${#dashboard_node_names[@]} -gt 0 ]; then
        common_logger "Generating Wazuh dashboard certificates."

        for i in "${!dashboard_node_names[@]}"; do
            dashboard_node_name="${dashboard_node_names[i]}"

            # Validate node name
            if ! cert_sanitizeNodeName "${dashboard_node_name}"; then
                common_logger -e "Invalid dashboard node name: ${dashboard_node_name}"
                exit 1
            fi

            cert_generateCertificateconfiguration "${dashboard_node_name}" "${dashboard_node_ips[i]}"
            common_logger -d "Creating the Wazuh dashboard tmp key pair."
            cert_executeAndValidate openssl req -new -nodes -newkey rsa:4096 -keyout "${cert_tmp_path}/${dashboard_node_name}-key.pem" -out "${cert_tmp_path}/${dashboard_node_name}.csr" -config "${cert_tmp_path}/${dashboard_node_name}.conf"
            common_logger -d "Creating the Wazuh dashboard certificates."
            cert_executeAndValidate openssl x509 -req -in "${cert_tmp_path}/${dashboard_node_name}.csr" -CA "${cert_tmp_path}/root-ca.pem" -CAkey "${cert_tmp_path}/root-ca.key" -CAcreateserial -out "${cert_tmp_path}/${dashboard_node_name}.pem" -extfile "${cert_tmp_path}/${dashboard_node_name}.conf" -extensions v3_req -days 1825
        done
    else
        return 1
    fi

}
function cert_generateRootCAcertificate() {

    common_logger "Generating the root certificate."

    # Validate cert_tmp_path
    if ! cert_validatePath "${cert_tmp_path}" "directory"; then
        common_logger -e "Invalid certificate temporary path."
        exit 1
    fi

    cert_executeAndValidate openssl req -x509 -new -nodes -newkey rsa:4096 -keyout "${cert_tmp_path}/root-ca.key" -out "${cert_tmp_path}/root-ca.pem" -batch -subj '/OU=SOC/O=CISO/L=Scottsdale/' -days 1825

}
function cert_parseYaml() {

    local prefix=$2
    local separator=${3:-_}
    local indexfix
    # Detect awk flavor
    if awk --version 2>&1 | grep -q "GNU Awk" ; then
    # GNU Awk detected
    indexfix=-1
    elif awk -Wv 2>&1 | grep -q "mawk" ; then
    # mawk detected
    indexfix=0
    fi

    local s='[[:space:]]*' sm='[ \t]*' w='[a-zA-Z0-9_]*' fs=${fs:-$(echo @|tr @ '\034')} i=${i:-  }
    cat $1 2>/dev/null | \
    awk -F$fs "{multi=0;
        if(match(\$0,/$sm\|$sm$/)){multi=1; sub(/$sm\|$sm$/,\"\");}
        if(match(\$0,/$sm>$sm$/)){multi=2; sub(/$sm>$sm$/,\"\");}
        while(multi>0){
            str=\$0; gsub(/^$sm/,\"\", str);
            indent=index(\$0,str);
            indentstr=substr(\$0, 0, indent+$indexfix) \"$i\";
            obuf=\$0;
            getline;
            while(index(\$0,indentstr)){
                obuf=obuf substr(\$0, length(indentstr)+1);
                if (multi==1){obuf=obuf \"\\\\n\";}
                if (multi==2){
                    if(match(\$0,/^$sm$/))
                        obuf=obuf \"\\\\n\";
                        else obuf=obuf \" \";
                }
                getline;
            }
            sub(/$sm$/,\"\",obuf);
            print obuf;
            multi=0;
            if(match(\$0,/$sm\|$sm$/)){multi=1; sub(/$sm\|$sm$/,\"\");}
            if(match(\$0,/$sm>$sm$/)){multi=2; sub(/$sm>$sm$/,\"\");}
        }
    print}" | \
    sed  -e "s|^\($s\)?|\1-|" \
        -ne "s|^$s#.*||;s|$s#[^\"']*$||;s|^\([^\"'#]*\)#.*|\1|;t1;t;:1;s|^$s\$||;t2;p;:2;d" | \
    sed -ne "s|,$s\]$s\$|]|" \
        -e ":1;s|^\($s\)\($w\)$s:$s\(&$w\)\?$s\[$s\(.*\)$s,$s\(.*\)$s\]|\1\2: \3[\4]\n\1$i- \5|;t1" \
        -e "s|^\($s\)\($w\)$s:$s\(&$w\)\?$s\[$s\(.*\)$s\]|\1\2: \3\n\1$i- \4|;" \
        -e ":2;s|^\($s\)-$s\[$s\(.*\)$s,$s\(.*\)$s\]|\1- [\2]\n\1$i- \3|;t2" \
        -e "s|^\($s\)-$s\[$s\(.*\)$s\]|\1-\n\1$i- \2|;p" | \
    sed -ne "s|,$s}$s\$|}|" \
        -e ":1;s|^\($s\)-$s{$s\(.*\)$s,$s\($w\)$s:$s\(.*\)$s}|\1- {\2}\n\1$i\3: \4|;t1" \
        -e "s|^\($s\)-$s{$s\(.*\)$s}|\1-\n\1$i\2|;" \
        -e ":2;s|^\($s\)\($w\)$s:$s\(&$w\)\?$s{$s\(.*\)$s,$s\($w\)$s:$s\(.*\)$s}|\1\2: \3 {\4}\n\1$i\5: \6|;t2" \
        -e "s|^\($s\)\($w\)$s:$s\(&$w\)\?$s{$s\(.*\)$s}|\1\2: \3\n\1$i\4|;p" | \
    sed  -e "s|^\($s\)\($w\)$s:$s\(&$w\)\(.*\)|\1\2:\4\n\3|" \
        -e "s|^\($s\)-$s\(&$w\)\(.*\)|\1- \3\n\2|" | \
    sed -ne "s|^\($s\):|\1|" \
        -e "s|^\($s\)\(---\)\($s\)||" \
        -e "s|^\($s\)\(\.\.\.\)\($s\)||" \
        -e "s|^\($s\)-$s[\"']\(.*\)[\"']$s\$|\1$fs$fs\2|p;t" \
        -e "s|^\($s\)\($w\)$s:$s[\"']\(.*\)[\"']$s\$|\1$fs\2$fs\3|p;t" \
        -e "s|^\($s\)-$s\(.*\)$s\$|\1$fs$fs\2|" \
        -e "s|^\($s\)\($w\)$s:$s[\"']\?\(.*\)$s\$|\1$fs\2$fs\3|" \
        -e "s|^\($s\)[\"']\?\([^&][^$fs]\+\)[\"']$s\$|\1$fs$fs$fs\2|" \
        -e "s|^\($s\)[\"']\?\([^&][^$fs]\+\)$s\$|\1$fs$fs$fs\2|" \
        -e "s|$s\$||p" | \
    awk -F$fs "{
        gsub(/\t/,\"        \",\$1);
        gsub(\"name: \", \"\");
        if(NF>3){if(value!=\"\"){value = value \" \";}value = value  \$4;}
        else {
        if(match(\$1,/^&/)){anchor[substr(\$1,2)]=full_vn;getline};
        indent = length(\$1)/length(\"$i\");
        vname[indent] = \$2;
        value= \$3;
        for (i in vname) {if (i > indent) {delete vname[i]; idx[i]=0}}
        if(length(\$2)== 0){  vname[indent]= ++idx[indent] };
        vn=\"\"; for (i=0; i<indent; i++) { vn=(vn)(vname[i])(\"$separator\")}
        vn=\"$prefix\" vn;
        full_vn=vn vname[indent];
        if(vn==\"$prefix\")vn=\"$prefix$separator\";
        if(vn==\"_\")vn=\"__\";
        }
        assignment[full_vn]=value;
        if(!match(assignment[vn], full_vn))assignment[vn]=assignment[vn] \" \" full_vn;
        if(match(value,/^\*/)){
            ref=anchor[substr(value,2)];
            if(length(ref)==0){
            printf(\"%s=\\\"%s\\\"\n\", full_vn, value);
            } else {
            for(val in assignment){
                if((length(ref)>0)&&index(val, ref)==1){
                    tmpval=assignment[val];
                    sub(ref,full_vn,val);
                if(match(val,\"$separator\$\")){
                    gsub(ref,full_vn,tmpval);
                } else if (length(tmpval) > 0) {
                    printf(\"%s=\\\"%s\\\"\n\", val, tmpval);
                }
                assignment[val]=tmpval;
                }
            }
        }
    } else if (length(value) > 0) {
        printf(\"%s=\\\"%s\\\"\n\", full_vn, value);
    }
    }END{
        for(val in assignment){
            if(match(val,\"$separator\$\"))
                printf(\"%s=\\\"%s\\\"\n\", val, assignment[val]);
        }
    }"

}
function cert_checkPrivateIp() {

    local ip=$1
    common_logger -d "Checking if ${ip} is private."

    # Check private IPv4 ranges
    if [[ $ip =~ ^10\.|^192\.168\.|^172\.(1[6-9]|2[0-9]|3[0-1])\.|^(127\.) ]]; then
        return 0
    fi

    # Check private IPv6 ranges (fc00::/7 prefix)
    if [[ $ip =~ ^fc ]]; then
        return 0
    fi

    return 1

}
function cert_readConfig() {

    common_logger -d "Reading configuration file."

    if [ -f "${config_file}" ]; then
        if [ ! -s "${config_file}" ]; then
            common_logger -e "File ${config_file} is empty"
            exit 1
        fi

        # Convert CRLF to LF
        if ! cert_convertCRLFtoLF "${config_file}"; then
            common_logger -e "Failed to convert configuration file from CRLF to LF: ${config_file}"
            exit 1
        fi

        # Read node names and IPs - use mapfile/readarray for safer array assignment
        mapfile -t indexer_node_names < <(cert_parseYaml "${config_file}" | grep -E "nodes[_]+indexer[_]+[0-9]+=" | cut -d = -f 2 | tr -d '"' | tr -d "'")
        mapfile -t server_node_names < <(cert_parseYaml "${config_file}" | grep -E "nodes[_]+server[_]+[0-9]+=" | cut -d = -f 2 | tr -d '"' | tr -d "'")
        mapfile -t dashboard_node_names < <(cert_parseYaml "${config_file}" | grep -E "nodes[_]+dashboard[_]+[0-9]+=" | cut -d = -f 2 | tr -d '"' | tr -d "'")
        mapfile -t indexer_node_ips < <(cert_parseYaml "${config_file}" | grep -E "nodes[_]+indexer[_]+[0-9]+[_]+ip=" | cut -d = -f 2 | tr -d '"' | tr -d "'")
        mapfile -t server_node_ips < <(cert_parseYaml "${config_file}" | grep -E "nodes[_]+server[_]+[0-9]+[_]+ip=" | cut -d = -f 2 | tr -d '"' | tr -d "'")
        mapfile -t dashboard_node_ips < <(cert_parseYaml "${config_file}" | grep -E "nodes[_]+dashboard[_]+[0-9]+[_]+ip=" | cut -d = -f 2 | tr -d '"' | tr -d "'")
        mapfile -t server_node_types < <(cert_parseYaml "${config_file}" | grep -E "nodes[_]+server[_]+[0-9]+[_]+node_type=" | cut -d = -f 2 | tr -d '"' | tr -d "'")

        number_server_ips=$(cert_parseYaml "${config_file}" | grep -o -E 'nodes[_]+server[_]+[0-9]+[_]+ip' | sort -u | wc -l)
        all_ips=("${indexer_node_ips[@]}" "${server_node_ips[@]}" "${dashboard_node_ips[@]}")

        # Validate all node names
        for name in "${indexer_node_names[@]}" "${server_node_names[@]}" "${dashboard_node_names[@]}"; do
            if ! cert_sanitizeNodeName "${name}"; then
                common_logger -e "Invalid node name found in config: ${name}"
                exit 1
            fi
        done

        for ip in "${all_ips[@]}"; do
            isIP=$(echo "${ip}" | grep -P "^[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}\.[0-9]{1,3}$")
            if [[ -n "${isIP}" ]]; then
                if ! cert_checkPrivateIp "$ip"; then
                    common_logger -e "The IP ${ip} is public."
                    exit 1
                fi
            fi
        done

        for i in $(seq 1 "${number_server_ips}"); do
            nodes_server="nodes[_]+server[_]+${i}[_]+ip"
            mapfile -t "server_node_ip_$i" < <(cert_parseYaml "${config_file}" | grep -E "${nodes_server}" | sed '/\./!d' | cut -d = -f 2 | sed -r 's/\s+//g' | tr -d '"' | tr -d "'")
        done

        unique_names=($(echo "${indexer_node_names[@]}" | tr ' ' '\n' | sort -u | tr '\n' ' '))
        if [ "${#unique_names[@]}" -ne "${#indexer_node_names[@]}" ]; then
            common_logger -e "Duplicated indexer node names."
            exit 1
        fi

        unique_ips=($(echo "${indexer_node_ips[@]}" | tr ' ' '\n' | sort -u | tr '\n' ' '))
        if [ "${#unique_ips[@]}" -ne "${#indexer_node_ips[@]}" ]; then
            common_logger -e "Duplicated indexer node ips."
            exit 1
        fi

        unique_names=($(echo "${server_node_names[@]}" | tr ' ' '\n' | sort -u | tr '\n' ' '))
        if [ "${#unique_names[@]}" -ne "${#server_node_names[@]}" ]; then
            common_logger -e "Duplicated Wazuh server node names."
            exit 1
        fi

        unique_ips=($(echo "${server_node_ips[@]}" | tr ' ' '\n' | sort -u | tr '\n' ' '))
        if [ "${#unique_ips[@]}" -ne "${#server_node_ips[@]}" ]; then
            common_logger -e "Duplicated Wazuh server node ips."
            exit 1
        fi

        unique_names=($(echo "${dashboard_node_names[@]}" | tr ' ' '\n' | sort -u | tr '\n' ' '))
        if [ "${#unique_names[@]}" -ne "${#dashboard_node_names[@]}" ]; then
            common_logger -e "Duplicated dashboard node names."
            exit 1
        fi

        unique_ips=($(echo "${dashboard_node_ips[@]}" | tr ' ' '\n' | sort -u | tr '\n' ' '))
        if [ "${#unique_ips[@]}" -ne "${#dashboard_node_ips[@]}" ]; then
            common_logger -e "Duplicated dashboard node ips."
            exit 1
        fi

        for i in "${server_node_types[@]}"; do
            if ! echo "$i" | grep -ioq master && ! echo "$i" | grep -ioq worker; then
                common_logger -e "Incorrect node_type $i must be master or worker"
                exit 1
            fi
        done

        if [ "${#server_node_names[@]}" -le 1 ]; then
            if [ "${#server_node_types[@]}" -ne 0 ]; then
                common_logger -e "The tag node_type can only be used with more than one Wazuh server."
                exit 1
            fi
        elif [ "${#server_node_names[@]}" -gt "${#server_node_types[@]}" ]; then
            common_logger -e "The tag node_type needs to be specified for all Wazuh server nodes."
            exit 1
        elif [ "${#server_node_names[@]}" -lt "${#server_node_types[@]}" ]; then
            common_logger -e "Found extra node_type tags."
            exit 1
        elif [ "$(grep -io master <<< "${server_node_types[*]}" | wc -l)" -ne 1 ]; then
            common_logger -e "Wazuh cluster needs a single master node."
            exit 1
        elif [ "$(grep -io worker <<< "${server_node_types[*]}" | wc -l)" -ne $(( ${#server_node_types[@]} - 1 )) ]; then
            common_logger -e "Incorrect number of workers."
            exit 1
        fi

        if [ "${#dashboard_node_names[@]}" -ne "${#dashboard_node_ips[@]}" ]; then
            common_logger -e "Different number of dashboard node names and IPs."
            exit 1
        fi

    else
        common_logger -e "No configuration file found."
        exit 1
    fi

}
function cert_setpermisions() {
    # Validate cert_tmp_path
    if ! cert_validatePath "${cert_tmp_path}" "directory"; then
        common_logger -e "Invalid certificate temporary path."
        return 1
    fi

    # Restrict the certificate directory and subdirectories to the owner only
    chmod 700 "${cert_tmp_path}" || return 1
    find "${cert_tmp_path}" -type d -exec chmod 700 {} \; || return 1
    # Restrict private key material to the owner only
    find "${cert_tmp_path}" -type f \( -name '*.key' -o -name '*.pem' \) -exec chmod 600 {} \; || return 1
    # Public certificate material can be readable by others
    find "${cert_tmp_path}" -type f \( -name '*.crt' -o -name '*.cer' -o -name '*.csr' \) -exec chmod 644 {} \; || return 1
}
function cert_convertCRLFtoLF() {
    local config_file_path="$1"

    # Validate config file path
    if ! cert_validatePath "${config_file_path}" "file"; then
        common_logger -e "Invalid config file path."
        return 1
    fi

    # Create secure temporary directory with mktemp
    local temp_dir
    temp_dir=$(mktemp -d -t wazuh-install-files.XXXXXXXXXX) || {
        common_logger -e "Failed to create temporary directory"
        return 1
    }

    # Set restrictive permissions
    chmod 700 "${temp_dir}"

    # Convert CRLF to LF
    tr -d '\015' < "${config_file_path}" > "${temp_dir}/new_config.yml"

    # Move back to original location
    mv "${temp_dir}/new_config.yml" "${config_file_path}"

    # Clean up temporary directory
    rm -rf "${temp_dir}"
}

# ------------ certMain.sh ------------ 
function getHelp() {

    echo -e ""
    echo -e "NAME"
    echo -e "        wazuh-cert-tool.sh - Manages the creation of certificates of the Wazuh components."
    echo -e ""
    echo -e "SYNOPSIS"
    echo -e "        wazuh-cert-tool.sh [OPTIONS]"
    echo -e ""
    echo -e "DESCRIPTION"
    echo -e "        -a,  --admin-certificates </path/to/root-ca.pem> </path/to/root-ca.key>"
    echo -e "                Creates the admin certificates, add root-ca.pem and root-ca.key."
    echo -e ""
    echo -e "        -A, --all </path/to/root-ca.pem> </path/to/root-ca.key>"
    echo -e "                Creates certificates specified in config.yml and admin certificates. Add a root-ca.pem and root-ca.key or leave it empty so a new one will be created."
    echo -e ""
    echo -e "        -ca, --root-ca-certificates"
    echo -e "                Creates the root-ca certificates."
    echo -e ""
    echo -e "        -v,  --verbose"
    echo -e "                Enables verbose mode."
    echo -e ""
    echo -e "        -wd,  --wazuh-dashboard-certificates </path/to/root-ca.pem> </path/to/root-ca.key>"
    echo -e "                Creates the Wazuh dashboard certificates, add root-ca.pem and root-ca.key."
    echo -e ""
    echo -e "        -wi,  --wazuh-indexer-certificates </path/to/root-ca.pem> </path/to/root-ca.key>"
    echo -e "                Creates the Wazuh indexer certificates, add root-ca.pem and root-ca.key."
    echo -e ""
    echo -e "        -ws,  --wazuh-server-certificates </path/to/root-ca.pem> </path/to/root-ca.key>"
    echo -e "                Creates the Wazuh server certificates, add root-ca.pem and root-ca.key."
    echo -e ""
    echo -e "        -tmp,  --cert_tmp_path </path/to/tmp_dir>"
    echo -e "                Modifies the default tmp directory (/tmp/wazuh-ceritificates) to the specified one."
    echo -e "                Must be used along with one of these options: -a, -A, -ca, -wi, -wd, -ws"
    echo -e ""

    exit 1

}
function main() {

    # Set a restrictive umask so new regular files default to 600 and directories to 700,
    # limiting access to the current user.
    umask 0077

    cert_checkOpenSSL

    if [ -n "${1}" ]; then
        while [ -n "${1}" ]
        do
            case "${1}" in
            "-a"|"--admin-certificates")
                if [[ -z "${2}" || -z "${3}" ]]; then
                    common_logger -e "Error on arguments. Probably missing </path/to/root-ca.pem> </path/to/root-ca.key> after -a|--admin-certificates"
                    getHelp
                    exit 1
                else
                    cadmin=1
                    rootca="${2}"
                    rootcakey="${3}"
                    shift 3
                fi
                ;;
            "-A"|"--all")
                if  [[ -n "${2}" && "${2}" != "-v" && "${2}" != "-tmp" ]]; then
                    # Validate that the user has entered the 2 files
                    if [[ -z ${3} ]]; then
                        if [[ ${2} == *".key" ]]; then
                            common_logger -e "You have not entered a root-ca.pem"
                            exit 1
                        else
                            common_logger -e "You have not entered a root-ca.key"
                            exit 1
                        fi
                    fi
                    all=1
                    rootca="${2}"
                    rootcakey="${3}"
                    shift 3
                else
                    all=1
                    shift 1
                fi
                ;;
            "-ca"|"--root-ca-certificate")
                ca=1
                shift 1
                ;;
            "-h"|"--help")
                getHelp
                ;;
            "-v"|"--verbose")
                debugEnabled=1
                shift 1
                ;;
            "-wd"|"--wazuh-dashboard-certificates")
                if [[ -z "${2}" || -z "${3}" ]]; then
                    common_logger -e "Error on arguments. Probably missing </path/to/root-ca.pem> </path/to/root-ca.key> after -wd|--wazuh-dashboard-certificates"
                    getHelp
                    exit 1
                else
                    cdashboard=1
                    rootca="${2}"
                    rootcakey="${3}"
                    shift 3
                fi
                ;;
            "-wi"|"--wazuh-indexer-certificates")
                if [[ -z "${2}" || -z "${3}" ]]; then
                    common_logger -e "Error on arguments. Probably missing </path/to/root-ca.pem> </path/to/root-ca.key> after -wi|--wazuh-indexer-certificates"
                    getHelp
                    exit 1
                else
                    cindexer=1
                    rootca="${2}"
                    rootcakey="${3}"
                    shift 3
                fi
                ;;
            "-ws"|"--wazuh-server-certificates")
                if [[ -z "${2}" || -z "${3}" ]]; then
                    common_logger -e "Error on arguments. Probably missing </path/to/root-ca.pem> </path/to/root-ca.key> after -ws|--wazuh-server-certificates"
                    getHelp
                    exit 1
                else
                    cserver=1
                    rootca="${2}"
                    rootcakey="${3}"
                    shift 3
                fi
                ;;
            "-tmp"|"--cert_tmp_path")
                if [[ -n "${3}" || ( "${cadmin}" == 1 || "${all}" == 1 || "${ca}" == 1 || "${cdashboard}" == 1 || "${cindexer}" == 1 || "${cserver}" == 1 ) ]]; then
                    if [[ -z "${2}" || ! "${2}" == /* ]]; then
                        common_logger -e "Error on arguments. Probably missing </path/to/tmp_dir> or path does not start with '/'."
                        getHelp
                        exit 1
                    else
                        cert_tmp_path="${2}"
                        shift 2
                    fi
                else
                    common_logger -e "Error: -tmp must be used along with one of these options: -a, -A, -ca, -wi, -wd, -ws"
                    getHelp
                    exit 1
                fi
                ;;
            *)
                echo "Unknow option: ${1}"
                getHelp
            esac
        done

        common_logger "Verbose logging redirected to ${logfile}"

        if [[ -d "${base_path}"/wazuh-certificates ]]; then
            if [ -n "$(ls -A "${base_path}"/wazuh-certificates)" ]; then
                common_logger -e "Directory wazuh-certificates already exists in the same path as the script. Please, remove the certs directory to create new certificates."
                exit 1
            fi
        fi

        # Validate and create secure temporary directory
        if ! cert_validatePath "${cert_tmp_path}" "directory"; then
            common_logger -e "Invalid temporary path: ${cert_tmp_path}"
            exit 1
        fi

        if [[ ! -d "${cert_tmp_path}" ]]; then
            # Create directory with secure permissions
            mkdir -p "${cert_tmp_path}"
            chmod 700 "${cert_tmp_path}"
        else
            # Ensure existing directory has secure permissions
            chmod 700 "${cert_tmp_path}"
        fi

        cert_readConfig

        if [ -n "${debugEnabled}" ]; then
            debug="2>&1 | tee -a ${logfile}"
        fi

        if [[ -n "${cadmin}" ]]; then
            cert_checkRootCA
            cert_generateAdmincertificate
            common_logger "Admin certificates created."
            cert_cleanFiles
            cert_setpermisions
            mv "${cert_tmp_path}" "${base_path}/wazuh-certificates"
        fi

        if [[ -n "${all}" ]]; then
            cert_checkRootCA
            cert_generateAdmincertificate
            common_logger "Admin certificates created."
            if cert_generateIndexercertificates; then
                common_logger "Wazuh indexer certificates created."
            fi
            if cert_generateFilebeatcertificates; then
                common_logger "Wazuh Filebeat certificates created."
            fi
            if cert_generateDashboardcertificates; then
                common_logger "Wazuh dashboard certificates created."
            fi
            cert_cleanFiles
            cert_setpermisions
            mv "${cert_tmp_path}" "${base_path}/wazuh-certificates"
        fi

        if [[ -n "${ca}" ]]; then
            cert_generateRootCAcertificate
            common_logger "Authority certificates created."
            cert_cleanFiles
            mv "${cert_tmp_path}" "${base_path}/wazuh-certificates"
        fi

        if [[ -n "${cindexer}" ]]; then
            if [ ${#indexer_node_names[@]} -gt 0 ]; then
                cert_checkRootCA
                cert_generateIndexercertificates
                common_logger "Wazuh indexer certificates created."
                cert_cleanFiles
                cert_setpermisions
                mv "${cert_tmp_path}" "${base_path}/wazuh-certificates"
            else
                common_logger -e "Indexer node not present in config.yml."
                exit 1
            fi
        fi

        if [[ -n "${cserver}" ]]; then
            if [ ${#server_node_names[@]} -gt 0 ]; then
                cert_checkRootCA
                cert_generateFilebeatcertificates
                common_logger "Wazuh Filebeat certificates created."
                cert_cleanFiles
                cert_setpermisions
                mv "${cert_tmp_path}" "${base_path}/wazuh-certificates"
            else
                common_logger -e "Server node not present in config.yml."
                exit 1
            fi
        fi

        if [[ -n "${cdashboard}" ]]; then
            if [ ${#dashboard_node_names[@]} -gt 0 ]; then
                cert_checkRootCA
                cert_generateDashboardcertificates
                common_logger "Wazuh dashboard certificates created."
                cert_cleanFiles
                cert_setpermisions
                mv "${cert_tmp_path}" "${base_path}/wazuh-certificates"
            else
                common_logger -e "Dashboard node not present in config.yml."
                exit 1
            fi
        fi

    else
        getHelp
    fi

}
# ------------ certVariables.sh ------------ 

function common_checkAptLock() {

    attempt=0
    seconds=30
    max_attempts=10

    while fuser "${apt_lockfile}" >/dev/null 2>&1 && [ "${attempt}" -lt "${max_attempts}" ]; do
        attempt=$((attempt+1))
        common_logger "Another process is using APT. Waiting for it to release the lock. Next retry in ${seconds} seconds (${attempt}/${max_attempts})"
        sleep "${seconds}"
    done

}
function common_logger() {

    now=$(date +'%d/%m/%Y %H:%M:%S')
    mtype="INFO:"
    debugLogger=
    nolog=
    if [ -n "${1}" ]; then
        while [ -n "${1}" ]; do
            case ${1} in
                "-e")
                    mtype="ERROR:"
                    shift 1
                    ;;
                "-w")
                    mtype="WARNING:"
                    shift 1
                    ;;
                "-d")
                    debugLogger=1
                    mtype="DEBUG:"
                    shift 1
                    ;;
                "-nl")
                    nolog=1
                    shift 1
                    ;;
                *)
                    message="${1}"
                    shift 1
                    ;;
            esac
        done
    fi

    if [ -z "${debugLogger}" ] || { [ -n "${debugLogger}" ] && [ -n "${debugEnabled}" ]; }; then
        if [ -z "${nolog}" ] && { [ "$EUID" -eq 0 ] || [[ "$(basename "$0")" =~ $cert_tool_script_name ]]; }; then
            printf "%s\n" "${now} ${mtype} ${message}" | tee -a ${logfile}
        else
            printf "%b\n" "${now} ${mtype} ${message}"
        fi
    fi

}
function common_checkRoot() {

    common_logger -d "Checking root permissions."
    if [ "$EUID" -ne 0 ]; then
        echo "This script must be run as root."
        exit 1;
    fi

}
function common_checkInstalled() {

    common_logger -d "Checking Wazuh installation."
    wazuh_installed=""
    indexer_installed=""
    filebeat_installed=""
    dashboard_installed=""

    if [ "${sys_type}" == "yum" ]; then
        rpm -q wazuh-manager --quiet && wazuh_installed=1
    elif [ "${sys_type}" == "apt-get" ]; then
        wazuh_installed=$(apt list --installed  2>/dev/null | grep wazuh-manager)
    fi

    if [ -d "/var/ossec" ]; then
        common_logger -d "There are Wazuh remaining files."
        wazuh_remaining_files=1
    fi

    if [ "${sys_type}" == "yum" ]; then
        rpm -q wazuh-indexer --quiet && indexer_installed=1

    elif [ "${sys_type}" == "apt-get" ]; then
        indexer_installed=$(apt list --installed 2>/dev/null | grep wazuh-indexer)
    fi

    if [ -d "/var/lib/wazuh-indexer/" ] || [ -d "/usr/share/wazuh-indexer" ] || [ -d "/etc/wazuh-indexer" ] || [ -f "${base_path}/search-guard-tlstool*" ]; then
        common_logger -d "There are Wazuh indexer remaining files."
        indexer_remaining_files=1
    fi

    if [ "${sys_type}" == "yum" ]; then
        rpm -q filebeat --quiet && filebeat_installed=1
    elif [ "${sys_type}" == "apt-get" ]; then
        filebeat_installed=$(apt list --installed  2>/dev/null | grep filebeat)
    fi

    if [ -d "/var/lib/filebeat/" ] || [ -d "/usr/share/filebeat" ] || [ -d "/etc/filebeat" ]; then
        common_logger -d "There are Filebeat remaining files."
        filebeat_remaining_files=1
    fi

    if [ "${sys_type}" == "yum" ]; then
        rpm -q wazuh-dashboard --quiet && dashboard_installed=1
    elif [ "${sys_type}" == "apt-get" ]; then
        dashboard_installed=$(apt list --installed  2>/dev/null | grep wazuh-dashboard)
    fi

    if [ -d "/var/lib/wazuh-dashboard/" ] || [ -d "/usr/share/wazuh-dashboard" ] || [ -d "/etc/wazuh-dashboard" ] || [ -d "/run/wazuh-dashboard/" ]; then
        common_logger -d "There are Wazuh dashboard remaining files."
        dashboard_remaining_files=1
    fi

}
function common_checkSystem() {

    if [ -n "$(command -v yum)" ]; then
        sys_type="yum"
        sep="-"
        common_logger -d "YUM package manager will be used."
    elif [ -n "$(command -v apt-get)" ]; then
        sys_type="apt-get"
        sep="="
        common_logger -d "APT package manager will be used."
    else
        common_logger -e "Couldn't find YUM or APT package manager. Try installing the one corresponding to your operating system and then, launch the installation assistant again."
        exit 1
    fi

}
function common_checkWazuhConfigYaml() {

    common_logger -d "Checking Wazuh YAML configuration file."
    filecorrect=$(cert_parseYaml "${config_file}" | grep -Ev '^#|^\s*$' | grep -Pzc "\A(\s*(nodes_indexer__name|nodes_indexer__ip|nodes_server__name|nodes_server__ip|nodes_server__node_type|nodes_dashboard__name|nodes_dashboard__ip)=.*?)+\Z")
    if [[ "${filecorrect}" -ne 1 ]]; then
        common_logger -e "The configuration file ${config_file} does not have a correct format."
        exit 1
    fi

}
function common_curl() {

    if [ -n "${curl_has_connrefused}" ]; then
        eval "curl $@ --retry-connrefused"
        e_code="${PIPESTATUS[0]}"
    else
        retries=0
        eval "curl $@"
        e_code="${PIPESTATUS[0]}"
        while [ "${e_code}" -eq 7 ] && [ "${retries}" -ne 12 ]; do
            retries=$((retries+1))
            sleep 5
            eval "curl $@"
            e_code="${PIPESTATUS[0]}"
        done
    fi
    return "${e_code}"

}
function common_remove_gpg_key() {

    common_logger -d "Removing GPG key from system."
    if [ "${sys_type}" == "yum" ]; then
        if { rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n' | grep "Wazuh"; } >/dev/null ; then
            key=$(rpm -q gpg-pubkey --qf '%{NAME}-%{VERSION}-%{RELEASE}\t%{SUMMARY}\n' | grep "Wazuh Signing Key" | awk '{print $1}' )
            rpm -e "${key}"
        else
            common_logger "Wazuh GPG key not found in the system"
            return 1
        fi
    elif [ "${sys_type}" == "apt-get" ]; then
        if [ -f "/usr/share/keyrings/wazuh.gpg" ]; then
            rm -rf "/usr/share/keyrings/wazuh.gpg" "${debug}"
        else
            common_logger "Wazuh GPG key not found in the system"
            return 1
        fi
    fi

}
function common_checkYumLock() {

    attempt=0
    seconds=30
    max_attempts=10

    while [ -f "${yum_lockfile}" ] && [ "${attempt}" -lt "${max_attempts}" ]; do
        attempt=$((attempt+1))
        common_logger "Another process is using YUM. Waiting for it to release the lock. Next retry in ${seconds} seconds (${attempt}/${max_attempts})"
        sleep "${seconds}"
    done

}

main "$@"
```
Create a configuration file named `config.yml` in the same directory as the script with the following content and replace the IP's values with the actual IP addresses:
```
nodes:
  # Wazuh indexer server nodes
  indexer:
    - name: wazuh1.indexer
      ip: wazuh1_indexer_ip
    - name: wazuh2.indexer
      ip: wazuh2_indexer_ip
    - name: wazuh3.indexer
      ip: wazuh3_indexer_ip
    - name: wazuh4.indexer
      ip: wazuh4_indexer_ip
    - name: wazuh5.indexer
      ip: wazuh5_indexer_ip

  # Wazuh server nodes
  # Use node_type only with more than one Wazuh manager
  server:
    - name: wazuh.master
      ip: wazuh_master_ip
      node_type: master
    - name: wazuh.worker1
      ip: wazuh_worker1_ip
      node_type: worker
    - name: wazuh.worker2
      ip: wazuh_worker2_ip
      node_type: worker

  # Wazuh dashboard node
  dashboard:
    - name: wazuh.dashboard
      ip: wazuh_dashboard_ip
```
use the following command to run the script and generate all certificates:
```
./wazuh-certs-tool.sh -A
```
You will find the generated certificates in the `wazuh-certificates` directory created in the same path as the script.
Distribute the certificates to the corresponding nodes and configure the Wazuh components to use them.

## 2 Wazuh indexers deployment
### 2.1 Wazuh indexer1 installation 
Please follow the steps below to install Wazuh indexer1 on the corresponding node.

In this guide we are using the path /data/Premium/workspace

Create a docker-compose.yml file with the following content:
```
# Wazuh App Copyright (C) 2017, Wazuh Inc. (License GPLv2)
services:
  wazuh1.indexer:
    image: socacr01.azurecr.io/wazuh-indexer:4.14.4
    container_name: ${WAZUH_INDEXER1_CONTAINER_NAME}
    hostname: ${WAZUH_INDEXER1_HOSTNAME}
    restart: always
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g"
      - "bootstrap.memory_lock=true"
      - AZURE_ACCOUNT=${AZURE_ACCOUNT}
      - AZURE_KEY=${AZURE_KEY}
    command: >
      bash -c '
        set -e;
        if ! /usr/share/wazuh-indexer/bin/opensearch-plugin list | grep -q repository-azure; then
          /usr/share/wazuh-indexer/bin/opensearch-plugin install --batch repository-azure;
        fi;
        if [ ! -f /usr/share/wazuh-indexer/config/opensearch.keystore ]; then
          /usr/share/wazuh-indexer/bin/opensearch-keystore create;
        fi;
        echo "$AZURE_ACCOUNT" | /usr/share/wazuh-indexer/bin/opensearch-keystore add -f --stdin azure.client.default.account;
        echo "$AZURE_KEY" | /usr/share/wazuh-indexer/bin/opensearch-keystore add -f --stdin azure.client.default.key;
        exec /entrypoint.sh
      '
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - ${WAZUH_INDEXER1_DATA_DIR}:/var/lib/wazuh-indexer
      - ${root_ca_pem}:/usr/share/wazuh-indexer/config/certs/root-ca.pem
      - ${wazuh1_indexer_key_pem}:/usr/share/wazuh-indexer/config/certs/wazuh1.indexer.key
      - ${wazuh1_indexer_pem}:/usr/share/wazuh-indexer/config/certs/wazuh1.indexer.pem
      - ${admin_pem}:/usr/share/wazuh-indexer/config/certs/admin.pem
      - ${admin_key_pem}:/usr/share/wazuh-indexer/config/certs/admin-key.pem
      - ./opensearch.yml:/usr/share/wazuh-indexer/config/opensearch.yml
      - ./internal_users.yml:/usr/share/wazuh-indexer/config/opensearch-security/internal_users.yml
```
Create a opensearch.yml file with the following content:
```
network.host: 0.0.0.0
network.publish_host: replace_with_your_indexer1_ip
node.name: wazuh1.indexer
node.roles:
  - cluster_manager
  - data
  - ingest
node.attr.temp: hot
cluster.initial_master_nodes:
  - wazuh1.indexer
  - wazuh2.indexer
  - wazuh3.indexer
cluster.name: "wazuh-cluster"
discovery.seed_hosts:
       - replace_with_your_indexer1_ip
       - replace_with_your_indexer2_ip
       - replace_with_your_indexer3_ip
       - replace_with_your_indexer4_ip
       - replace_with_your_indexer5_ip
path.data: /var/lib/wazuh-indexer
path.logs: /var/log/wazuh-indexer
plugins.security.ssl.http.pemcert_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh1.indexer.pem
plugins.security.ssl.http.pemkey_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh1.indexer.key
plugins.security.ssl.http.pemtrustedcas_filepath: ${OPENSEARCH_PATH_CONF}/certs/root-ca.pem
plugins.security.ssl.transport.pemcert_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh1.indexer.pem
plugins.security.ssl.transport.pemkey_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh1.indexer.key
plugins.security.ssl.transport.pemtrustedcas_filepath: ${OPENSEARCH_PATH_CONF}/certs/root-ca.pem
plugins.security.ssl.http.enabled: true
plugins.security.ssl.transport.enforce_hostname_verification: true
plugins.security.ssl.transport.resolve_hostname: false
plugins.security.ssl.http.enabled_ciphers:
 - "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
 - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
 - "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256"
 - "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384"
plugins.security.ssl.http.enabled_protocols:
 - "TLSv1.2"
plugins.security.authcz.admin_dn:
 - "CN=admin,OU=SOC,O=CISO,L=Scottsdale,C=US"
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.nodes_dn:
 - "CN=wazuh1.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh2.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh3.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh4.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh5.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=filebeat,OU=SOC,O=CISO,L=Scottsdale,C=US"
plugins.security.restapi.roles_enabled:
 - "all_access"
 - "security_rest_api_access"
plugins.security.allow_default_init_securityindex: true
cluster.routing.allocation.disk.threshold_enabled: false
compatibility.override_main_response_version: true
```
Create a internal_users.yml file with the following content:
```
---
# This is the internal user database
# The hash value is a bcrypt hash and can be generated with plugin/tools/hash.sh

_meta:
  type: "internalusers"
  config_version: 2

# Define your internal users here

## Demo users

admin:
  hash: "$2y$12$XiQR9Ju.ppzxljOh9bIrYuwv7R2FI263Vw/kZaqPMum56iIn5KYra"
  reserved: true
  backend_roles:
  - "admin"
  description: "Demo admin user"

kibanaserver:
  hash: "$2a$12$4AcgAt3xwOWadA5s5blL6ev39OXDNhmOesEoo33eZtrq2N0YrU3H."
  reserved: true
  description: "Demo kibanaserver user"

kibanaro:
  hash: "$2a$12$JJSXNfTowz7Uu5ttXfeYpeYE0arACvcwlPBStB1F.MI7f0U9Z4DGC"
  reserved: false
  backend_roles:
  - "kibanauser"
  - "readall"
  attributes:
    attribute1: "value1"
    attribute2: "value2"
    attribute3: "value3"
  description: "Demo kibanaro user"

logstash:
  hash: "$2a$12$u1ShR4l4uBS3Uv59Pa2y5.1uQuZBrZtmNfqB3iM/.jL0XoV9sghS2"
  reserved: false
  backend_roles:
  - "logstash"
  description: "Demo logstash user"

readall:
  hash: "$2a$12$ae4ycwzwvLtZxwZ82RmiEunBbIPiAmGZduBAjKN0TXdwQFtCwARz2"
  reserved: false
  backend_roles:
  - "readall"
  description: "Demo readall user"

snapshotrestore:
  hash: "$2y$12$DpwmetHKwgYnorbgdvORCenv4NAK8cPUg8AI6pxLCuWf/ALc0.v7W"
  reserved: false
  backend_roles:
  - "snapshotrestore"
  description: "Demo snapshotrestore user"
```
Create a .env file with the following content and replace the values with your own:
```
WAZUH_INDEXER1_CONTAINER_NAME=wazuh1.indexer
WAZUH_INDEXER1_HOSTNAME=wazuh1.indexer
WAZUH_INDEXER1_DATA_DIR=/data/Premium/data/
root_ca_pem=./certs/root-ca.pem
wazuh1_indexer_key_pem=./certs/wazuh1.indexer-key.pem
wazuh1_indexer_pem=./certs/wazuh1.indexer.pem
admin_pem=./certs/admin.pem
admin_key_pem=./certs/admin-key.pem
AZURE_ACCOUNT=your_azure_account_name
AZURE_KEY=your_azure_account_key
```
Create the certs directory and copy the generated certificates from the `wazuh-certificates` directory to the `certs` directory.
Azure account and key are used for the Azure repository plugin. The below steps are guide you to create an Azure storage account and get the account name and key.
1. Log in to the Azure portal.
2. In the left-hand menu, click on "Storage accounts" and then click on "Add" to create a new storage account.
3. Fill in the required information for your storage account, such as subscription, resource group, storage account name, region, and performance options. Click "Review + create" and then "Create" to create the storage account.
4. Once the storage account is created, navigate to the storage account's overview page.    
5. In the left-hand menu, click on "Access keys" under the "Security + networking" section.
6. You will see two access keys (key1 and key2) along with their corresponding connection strings. Copy the "Storage account name" and one of the "Key" values (key1 or key2) to use in your .env file.     

Once you have created the docker-compose.yml, opensearch.yml, internal_users.yml, and .env files, and copied the certificates to the certs directory, you can start the Wazuh indexer1 container by running the following command in the same directory as the docker-compose.yml file:
``` 
docker-compose up -d
``` 
Check the logs to ensure that the Wazuh indexer1 container is running correctly:
```
docker logs -f wazuh1.indexer
```
Once the container is running, you can move to the next step of deploying Wazuh indexer2.

### 2.2 Wazuh indexer2 installation
Please follow the steps below to install Wazuh indexer2 on the corresponding node.

In this guide we are using the path /data/Premium/workspace
Create a docker-compose.yml file with the following content:
```
# Wazuh App Copyright (C) 2017, Wazuh Inc. (License GPLv2)
services:
  wazuh2.indexer:
    image: socacr01.azurecr.io/wazuh-indexer:4.14.4
    container_name: ${WAZUH_INDEXER2_CONTAINER_NAME}
    hostname: ${WAZUH_INDEXER2_HOSTNAME}
    restart: always
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g"
      - "bootstrap.memory_lock=true"
      - AZURE_ACCOUNT=${AZURE_ACCOUNT}
      - AZURE_KEY=${AZURE_KEY}
    command: >
      bash -c '
        set -e;
        if ! /usr/share/wazuh-indexer/bin/opensearch-plugin list | grep -q repository-azure; then
          /usr/share/wazuh-indexer/bin/opensearch-plugin install --batch repository-azure;
        fi;
        if [ ! -f /usr/share/wazuh-indexer/config/opensearch.keystore ]; then
          /usr/share/wazuh-indexer/bin/opensearch-keystore create;
        fi;
        echo "$AZURE_ACCOUNT" | /usr/share/wazuh-indexer/bin/opensearch-keystore add -f --stdin azure.client.default.account;
        echo "$AZURE_KEY" | /usr/share/wazuh-indexer/bin/opensearch-keystore add -f --stdin azure.client.default.key;
        exec /entrypoint.sh
      '
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - ${WAZUH_INDEXER2_DATA_DIR}:/var/lib/wazuh-indexer
      - ${root_ca_pem}:/usr/share/wazuh-indexer/config/certs/root-ca.pem
      - ${wazuh2_indexer_key_pem}:/usr/share/wazuh-indexer/config/certs/wazuh2.indexer.key
      - ${wazuh2_indexer_pem}:/usr/share/wazuh-indexer/config/certs/wazuh2.indexer.pem
      - ${admin_pem}:/usr/share/wazuh-indexer/config/certs/admin.pem
      - ${admin_key_pem}:/usr/share/wazuh-indexer/config/certs/admin-key.pem
      - ./opensearch.yml:/usr/share/wazuh-indexer/config/opensearch.yml
      - ./internal_users.yml:/usr/share/wazuh-indexer/config/opensearch-security/internal_users.yml
```
Create a opensearch.yml file with the following content:
```
network.host: 0.0.0.0
network.publish_host: replace_with_your_indexer2_ip
node.name: wazuh2.indexer
node.roles:
  - cluster_manager
  - data
  - ingest
node.attr.temp: hot
cluster.initial_master_nodes:
  - wazuh1.indexer
  - wazuh2.indexer
  - wazuh3.indexer
cluster.name: "wazuh-cluster"
discovery.seed_hosts:
       - replace_with_your_indexer1_ip
       - replace_with_your_indexer2_ip
       - replace_with_your_indexer3_ip
       - replace_with_your_indexer4_ip
       - replace_with_your_indexer5_ip
path.data: /var/lib/wazuh-indexer
path.logs: /var/log/wazuh-indexer
plugins.security.ssl.http.pemcert_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh2.indexer.pem
plugins.security.ssl.http.pemkey_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh2.indexer.key
plugins.security.ssl.http.pemtrustedcas_filepath: ${OPENSEARCH_PATH_CONF}/certs/root-ca.pem
plugins.security.ssl.transport.pemcert_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh2.indexer.pem
plugins.security.ssl.transport.pemkey_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh2.indexer.key
plugins.security.ssl.transport.pemtrustedcas_filepath: ${OPENSEARCH_PATH_CONF}/certs/root-ca.pem
plugins.security.ssl.http.enabled: true
plugins.security.ssl.transport.enforce_hostname_verification: true
plugins.security.ssl.transport.resolve_hostname: false
plugins.security.ssl.http.enabled_ciphers:
 - "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
 - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
 - "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256"
 - "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384"
plugins.security.ssl.http.enabled_protocols:
 - "TLSv1.2"
plugins.security.authcz.admin_dn:
 - "CN=admin,OU=SOC,O=CISO,L=Scottsdale,C=US"
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.nodes_dn:
 - "CN=wazuh1.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh2.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh3.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh4.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh5.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=filebeat,OU=SOC,O=CISO,L=Scottsdale,C=US"
plugins.security.restapi.roles_enabled:
 - "all_access"
 - "security_rest_api_access"
plugins.security.allow_default_init_securityindex: true
cluster.routing.allocation.disk.threshold_enabled: false
compatibility.override_main_response_version: true
```
Create a internal_users.yml file with the following content:
```
---
# This is the internal user database
# The hash value is a bcrypt hash and can be generated with plugin/tools/hash.sh

_meta:
  type: "internalusers"
  config_version: 2

# Define your internal users here

## Demo users

admin:
  hash: "$2y$12$XiQR9Ju.ppzxljOh9bIrYuwv7R2FI263Vw/kZaqPMum56iIn5KYra"
  reserved: true
  backend_roles:
  - "admin"
  description: "Demo admin user"

kibanaserver:
  hash: "$2a$12$4AcgAt3xwOWadA5s5blL6ev39OXDNhmOesEoo33eZtrq2N0YrU3H."
  reserved: true
  description: "Demo kibanaserver user"

kibanaro:
  hash: "$2a$12$JJSXNfTowz7Uu5ttXfeYpeYE0arACvcwlPBStB1F.MI7f0U9Z4DGC"
  reserved: false
  backend_roles:
  - "kibanauser"
  - "readall"
  attributes:
    attribute1: "value1"
    attribute2: "value2"
    attribute3: "value3"
  description: "Demo kibanaro user"

logstash:
  hash: "$2a$12$u1ShR4l4uBS3Uv59Pa2y5.1uQuZBrZtmNfqB3iM/.jL0XoV9sghS2"
  reserved: false
  backend_roles:
  - "logstash"
  description: "Demo logstash user"

readall:
  hash: "$2a$12$ae4ycwzwvLtZxwZ82RmiEunBbIPiAmGZduBAjKN0TXdwQFtCwARz2"
  reserved: false
  backend_roles:
  - "readall"
  description: "Demo readall user"

snapshotrestore:
  hash: "$2y$12$DpwmetHKwgYnorbgdvORCenv4NAK8cPUg8AI6pxLCuWf/ALc0.v7W"
  reserved: false
  backend_roles:
  - "snapshotrestore"
  description: "Demo snapshotrestore user"
```
Create a .env file with the following content and replace the values with your own:
```
WAZUH_INDEXER2_CONTAINER_NAME=wazuh2.indexer
WAZUH_INDEXER2_HOSTNAME=wazuh2.indexer
WAZUH_INDEXER2_DATA_DIR=/data/Premium/data
root_ca_pem=./certs/root-ca.pem
wazuh2_indexer_pem=./certs/wazuh2.indexer.pem
wazuh2_indexer_key_pem=./certs/wazuh2.indexer-key.pem
admin_pem=./certs/admin.pem
admin_key_pem=./certs/admin-key.pem
AZURE_ACCOUNT=your_azure_account_name
AZURE_KEY=your_azure_account_key
```
Once you have created the docker-compose.yml, opensearch.yml, internal_users.yml, and .env files, and copied the certificates to the certs directory, you can start the Wazuh indexer2 container by running the following command in the same directory as the docker-compose.yml file:
```
docker-compose up -d
```
Check the logs to ensure that the Wazuh indexer2 container is running correctly:
```
docker logs -f wazuh2.indexer
```
Once the container is running, you can move to the next step of deploying Wazuh indexer3.

### 2.3 Wazuh indexer3 installation

Please follow the steps below to install Wazuh indexer3 on the corresponding node.
In this guide we are using the path /data/Premium/workspace
Create a docker-compose.yml file with the following content:    
```
# Wazuh App Copyright (C) 2017, Wazuh Inc. (License GPLv2)
services:
  wazuh3.indexer:
    image: socacr01.azurecr.io/wazuh-indexer:4.14.4
    container_name: ${WAZUH_INDEXER3_CONTAINER_NAME}
    hostname: ${WAZUH_INDEXER3_HOSTNAME}
    restart: always
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g"
      - "bootstrap.memory_lock=true"
      - AZURE_ACCOUNT=${AZURE_ACCOUNT}
      - AZURE_KEY=${AZURE_KEY}
    command: >
      bash -c '
        set -e;
        if ! /usr/share/wazuh-indexer/bin/opensearch-plugin list | grep -q repository-azure; then
          /usr/share/wazuh-indexer/bin/opensearch-plugin install --batch repository-azure;
        fi;
        if [ ! -f /usr/share/wazuh-indexer/config/opensearch.keystore ]; then
          /usr/share/wazuh-indexer/bin/opensearch-keystore create;
        fi;
        echo "$AZURE_ACCOUNT" | /usr/share/wazuh-indexer/bin/opensearch-keystore add -f --stdin azure.client.default.account;
        echo "$AZURE_KEY" | /usr/share/wazuh-indexer/bin/opensearch-keystore add -f --stdin azure.client.default.key;
        exec /entrypoint.sh
      '
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - ${WAZUH_INDEXER3_DATA_DIR}:/var/lib/wazuh-indexer
      - ${root_ca_pem}:/usr/share/wazuh-indexer/config/certs/root-ca.pem
      - ${wazuh3_indexer_key_pem}:/usr/share/wazuh-indexer/config/certs/wazuh3.indexer.key
      - ${wazuh3_indexer_pem}:/usr/share/wazuh-indexer/config/certs/wazuh3.indexer.pem
      - ${admin_pem}:/usr/share/wazuh-indexer/config/certs/admin.pem
      - ${admin_key_pem}:/usr/share/wazuh-indexer/config/certs/admin-key.pem
      - ./opensearch.yml:/usr/share/wazuh-indexer/config/opensearch.yml
      - ./internal_users.yml:/usr/share/wazuh-indexer/config/opensearch-security/internal_users.yml
```
Create a opensearch.yml file with the following content:
```
network.host: 0.0.0.0
network.publish_host: replace_with_your_indexer3_ip
node.name: wazuh3.indexer
node.roles:
  - cluster_manager
  - data
  - ingest
node.attr.temp: hot
cluster.initial_master_nodes:
  - wazuh1.indexer
  - wazuh2.indexer
  - wazuh3.indexer
cluster.name: "wazuh-cluster"
discovery.seed_hosts:
       - replace_with_your_indexer1_ip
       - replace_with_your_indexer2_ip
       - replace_with_your_indexer3_ip
       - replace_with_your_indexer4_ip
       - replace_with_your_indexer5_ip
path.data: /var/lib/wazuh-indexer
path.logs: /var/log/wazuh-indexer
plugins.security.ssl.http.pemcert_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh3.indexer.pem
plugins.security.ssl.http.pemkey_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh3.indexer.key
plugins.security.ssl.http.pemtrustedcas_filepath: ${OPENSEARCH_PATH_CONF}/certs/root-ca.pem
plugins.security.ssl.transport.pemcert_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh3.indexer.pem
plugins.security.ssl.transport.pemkey_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh3.indexer.key
plugins.security.ssl.transport.pemtrustedcas_filepath: ${OPENSEARCH_PATH_CONF}/certs/root-ca.pem
plugins.security.ssl.http.enabled: true
plugins.security.ssl.transport.enforce_hostname_verification: true
plugins.security.ssl.transport.resolve_hostname: false
plugins.security.ssl.http.enabled_ciphers:
 - "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
 - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
 - "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256"
 - "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384"
plugins.security.ssl.http.enabled_protocols:
 - "TLSv1.2"
plugins.security.authcz.admin_dn:
 - "CN=admin,OU=SOC,O=CISO,L=Scottsdale,C=US"
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.nodes_dn:
 - "CN=wazuh1.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh2.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh3.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh4.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh5.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=filebeat,OU=SOC,O=CISO,L=Scottsdale,C=US"
plugins.security.restapi.roles_enabled:
 - "all_access"
 - "security_rest_api_access"
plugins.security.allow_default_init_securityindex: true
cluster.routing.allocation.disk.threshold_enabled: false
compatibility.override_main_response_version: true
```
Create a internal_users.yml file with the following content:
```
---
# This is the internal user database
# The hash value is a bcrypt hash and can be generated with plugin/tools/hash.sh

_meta:
  type: "internalusers"
  config_version: 2

# Define your internal users here

## Demo users

admin:
  hash: "$2y$12$XiQR9Ju.ppzxljOh9bIrYuwv7R2FI263Vw/kZaqPMum56iIn5KYra"
  reserved: true
  backend_roles:
  - "admin"
  description: "Demo admin user"

kibanaserver:
  hash: "$2a$12$4AcgAt3xwOWadA5s5blL6ev39OXDNhmOesEoo33eZtrq2N0YrU3H."
  reserved: true
  description: "Demo kibanaserver user"

kibanaro:
  hash: "$2a$12$JJSXNfTowz7Uu5ttXfeYpeYE0arACvcwlPBStB1F.MI7f0U9Z4DGC"
  reserved: false
  backend_roles:
  - "kibanauser"
  - "readall"
  attributes:
    attribute1: "value1"
    attribute2: "value2"
    attribute3: "value3"
  description: "Demo kibanaro user"

logstash:
  hash: "$2a$12$u1ShR4l4uBS3Uv59Pa2y5.1uQuZBrZtmNfqB3iM/.jL0XoV9sghS2"
  reserved: false
  backend_roles:
  - "logstash"
  description: "Demo logstash user"

readall:
  hash: "$2a$12$ae4ycwzwvLtZxwZ82RmiEunBbIPiAmGZduBAjKN0TXdwQFtCwARz2"
  reserved: false
  backend_roles:
  - "readall"
  description: "Demo readall user"

snapshotrestore:
  hash: "$2y$12$DpwmetHKwgYnorbgdvORCenv4NAK8cPUg8AI6pxLCuWf/ALc0.v7W"
  reserved: false
  backend_roles:
  - "snapshotrestore"
  description: "Demo snapshotrestore user"
```
Create a .env file with the following content and replace the values with your own:
```
WAZUH_INDEXER3_CONTAINER_NAME=wazuh3.indexer
WAZUH_INDEXER3_HOSTNAME=wazuh3.indexer
WAZUH_INDEXER3_DATA_DIR=/data/Premium/data
root_ca_pem=./certs/root-ca.pem
wazuh3_indexer_pem=./certs/wazuh3.indexer.pem
wazuh3_indexer_key_pem=./certs/wazuh3.indexer-key.pem
admin_pem=./certs/admin.pem
admin_key_pem=./certs/admin-key.pem
AZURE_ACCOUNT=your_azure_account_name
AZURE_KEY=your_azure_account_key    
```
Once you have created the docker-compose.yml, opensearch.yml, internal_users.yml, and .env files, and copied the certificates to the certs directory, you can start the Wazuh indexer3 container by running the following command in the same directory as the docker-compose.yml file:
``` 
docker-compose up -d
```
Check the logs to ensure that the Wazuh indexer3 container is running correctly:
```
docker logs -f wazuh3.indexer
```
Once the container is running, you can move to the next step of deploying Wazuh indexer4.

### 2.4 Wazuh indexer4 installation
Please follow the steps below to install Wazuh indexer4 on the corresponding node.  

In this guide we are using the path /data/Premium/workspace
Create a docker-compose.yml file with the following content:    
```
# Wazuh App Copyright (C) 2017, Wazuh Inc. (License GPLv2)
services:
  wazuh4.indexer:
    image: socacr01.azurecr.io/wazuh-indexer:4.14.4
    container_name: ${WAZUH_INDEXER4_CONTAINER_NAME}
    hostname: ${WAZUH_INDEXER4_HOSTNAME}
    restart: always
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g"
      - "bootstrap.memory_lock=true"
      - AZURE_ACCOUNT=${AZURE_ACCOUNT}
      - AZURE_KEY=${AZURE_KEY}
    command: >
      bash -c '
        set -e;
        if ! /usr/share/wazuh-indexer/bin/opensearch-plugin list | grep -q repository-azure; then
          /usr/share/wazuh-indexer/bin/opensearch-plugin install --batch repository-azure;
        fi;
        if [ ! -f /usr/share/wazuh-indexer/config/opensearch.keystore ]; then
          /usr/share/wazuh-indexer/bin/opensearch-keystore create;
        fi;
        echo "$AZURE_ACCOUNT" | /usr/share/wazuh-indexer/bin/opensearch-keystore add -f --stdin azure.client.default.account;
        echo "$AZURE_KEY" | /usr/share/wazuh-indexer/bin/opensearch-keystore add -f --stdin azure.client.default.key;
        exec /entrypoint.sh
      '
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - ${WAZUH_INDEXER4_DATA_DIR}:/var/lib/wazuh-indexer
      - ${root_ca_pem}:/usr/share/wazuh-indexer/config/certs/root-ca.pem
      - ${wazuh4_indexer_key_pem}:/usr/share/wazuh-indexer/config/certs/wazuh4.indexer.key
      - ${wazuh4_indexer_pem}:/usr/share/wazuh-indexer/config/certs/wazuh4.indexer.pem
      - ${admin_pem}:/usr/share/wazuh-indexer/config/certs/admin.pem
      - ${admin_key_pem}:/usr/share/wazuh-indexer/config/certs/admin-key.pem
      - ./opensearch.yml:/usr/share/wazuh-indexer/config/opensearch.yml
      - ./internal_users.yml:/usr/share/wazuh-indexer/config/opensearch-security/internal_users.yml
```
create a opensearch.yml file with the following content:
```
network.host: 0.0.0.0
network.publish_host: replace_with_your_indexer4_ip
node.name: wazuh4.indexer
node.roles:
  - cluster_manager
  - data
node.attr.temp: warm
cluster.name: "wazuh-cluster"
discovery.seed_hosts:
       - replace_with_your_indexer1_ip
       - replace_with_your_indexer2_ip
       - replace_with_your_indexer3_ip
       - replace_with_your_indexer4_ip
       - replace_with_your_indexer5_ip
path.data: /var/lib/wazuh-indexer
path.logs: /var/log/wazuh-indexer
plugins.security.ssl.http.pemcert_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh4.indexer.pem
plugins.security.ssl.http.pemkey_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh4.indexer.key
plugins.security.ssl.http.pemtrustedcas_filepath: ${OPENSEARCH_PATH_CONF}/certs/root-ca.pem
plugins.security.ssl.transport.pemcert_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh4.indexer.pem
plugins.security.ssl.transport.pemkey_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh4.indexer.key
plugins.security.ssl.transport.pemtrustedcas_filepath: ${OPENSEARCH_PATH_CONF}/certs/root-ca.pem
plugins.security.ssl.http.enabled: true
plugins.security.ssl.transport.enforce_hostname_verification: true
plugins.security.ssl.transport.resolve_hostname: false
plugins.security.ssl.http.enabled_ciphers:
 - "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
 - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
 - "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256"
 - "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384"
plugins.security.ssl.http.enabled_protocols:
 - "TLSv1.2"
plugins.security.authcz.admin_dn:
 - "CN=admin,OU=SOC,O=CISO,L=Scottsdale,C=US"
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.nodes_dn:
 - "CN=wazuh1.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh2.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh3.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh4.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh5.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=filebeat,OU=SOC,O=CISO,L=Scottsdale,C=US"
plugins.security.restapi.roles_enabled:
 - "all_access"
 - "security_rest_api_access"
plugins.security.allow_default_init_securityindex: true
cluster.routing.allocation.disk.threshold_enabled: false
compatibility.override_main_response_version: true
```

Create a internal_users.yml file with the following content:
```
---
# This is the internal user database
# The hash value is a bcrypt hash and can be generated with plugin/tools/hash.sh

_meta:
  type: "internalusers"
  config_version: 2

# Define your internal users here

## Demo users

admin:
  hash: "$2y$12$XiQR9Ju.ppzxljOh9bIrYuwv7R2FI263Vw/kZaqPMum56iIn5KYra"
  reserved: true
  backend_roles:
  - "admin"
  description: "Demo admin user"

kibanaserver:
  hash: "$2a$12$4AcgAt3xwOWadA5s5blL6ev39OXDNhmOesEoo33eZtrq2N0YrU3H."
  reserved: true
  description: "Demo kibanaserver user"

kibanaro:
  hash: "$2a$12$JJSXNfTowz7Uu5ttXfeYpeYE0arACvcwlPBStB1F.MI7f0U9Z4DGC"
  reserved: false
  backend_roles:
  - "kibanauser"
  - "readall"
  attributes:
    attribute1: "value1"
    attribute2: "value2"
    attribute3: "value3"
  description: "Demo kibanaro user"

logstash:
  hash: "$2a$12$u1ShR4l4uBS3Uv59Pa2y5.1uQuZBrZtmNfqB3iM/.jL0XoV9sghS2"
  reserved: false
  backend_roles:
  - "logstash"
  description: "Demo logstash user"

readall:
  hash: "$2a$12$ae4ycwzwvLtZxwZ82RmiEunBbIPiAmGZduBAjKN0TXdwQFtCwARz2"
  reserved: false
  backend_roles:
  - "readall"
  description: "Demo readall user"

snapshotrestore:
  hash: "$2y$12$DpwmetHKwgYnorbgdvORCenv4NAK8cPUg8AI6pxLCuWf/ALc0.v7W"
  reserved: false
  backend_roles:
  - "snapshotrestore"
  description: "Demo snapshotrestore user"
```
Create a .env file with the following content and replace the values with your own:
```
WAZUH_INDEXER4_CONTAINER_NAME=wazuh4.indexer
WAZUH_INDEXER4_HOSTNAME=wazuh4.indexer
WAZUH_INDEXER4_DATA_DIR=/data/Premium/data/
root_ca_pem=./certs/root-ca.pem
wazuh4_indexer_pem=./certs/wazuh4.indexer.pem
wazuh4_indexer_key_pem=./certs/wazuh4.indexer-key.pem
admin_pem=./certs/admin.pem
admin_key_pem=./certs/admin-key.pem
AZURE_ACCOUNT=your_azure_account_name
AZURE_KEY=your_azure_account_key
```
Once you have created the docker-compose.yml, opensearch.yml, internal_users.yml, and .env files, and copied the certificates to the certs directory, you can start the Wazuh indexer4 container by running the following command in the same directory as the docker-compose.yml file:
``` 
docker-compose up -d
```
Check the logs to ensure that the Wazuh indexer4 container is running correctly:
```
docker logs -f wazuh4.indexer
```
Once the container is running, you can move to the next step of deploying Wazuh indexer5.

### 2.5 Wazuh indexer5 installation
Please follow the steps below to install Wazuh indexer5 on the corresponding node.
In this guide we are using the path /data/Premium/workspace
Create a docker-compose.yml file with the following content:
```
# Wazuh App Copyright (C) 2017, Wazuh Inc. (License GPLv2)
services:
  wazuh5.indexer:
    image: socacr01.azurecr.io/wazuh-indexer:4.14.4
    container_name: ${WAZUH_INDEXER5_CONTAINER_NAME}
    hostname: ${WAZUH_INDEXER5_HOSTNAME}
    restart: always
    ports:
      - "9200:9200"
      - "9300:9300"
    environment:
      - "OPENSEARCH_JAVA_OPTS=-Xms4g -Xmx4g"
      - "bootstrap.memory_lock=true"
      - AZURE_ACCOUNT=${AZURE_ACCOUNT}
      - AZURE_KEY=${AZURE_KEY}
    command: >
      bash -c '
        set -e;
        if ! /usr/share/wazuh-indexer/bin/opensearch-plugin list | grep -q repository-azure; then
          /usr/share/wazuh-indexer/bin/opensearch-plugin install --batch repository-azure;
        fi;
        if [ ! -f /usr/share/wazuh-indexer/config/opensearch.keystore ]; then
          /usr/share/wazuh-indexer/bin/opensearch-keystore create;
        fi;
        echo "$AZURE_ACCOUNT" | /usr/share/wazuh-indexer/bin/opensearch-keystore add -f --stdin azure.client.default.account;
        echo "$AZURE_KEY" | /usr/share/wazuh-indexer/bin/opensearch-keystore add -f --stdin azure.client.default.key;
        exec /entrypoint.sh
      '
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 65536
        hard: 65536
    volumes:
      - ${WAZUH_INDEXER5_DATA_DIR}:/var/lib/wazuh-indexer
      - ${root_ca_pem}:/usr/share/wazuh-indexer/config/certs/root-ca.pem
      - ${wazuh5_indexer_key_pem}:/usr/share/wazuh-indexer/config/certs/wazuh5.indexer.key
      - ${wazuh5_indexer_pem}:/usr/share/wazuh-indexer/config/certs/wazuh5.indexer.pem
      - ${admin_pem}:/usr/share/wazuh-indexer/config/certs/admin.pem
      - ${admin_key_pem}:/usr/share/wazuh-indexer/config/certs/admin-key.pem
      - ./opensearch.yml:/usr/share/wazuh-indexer/config/opensearch.yml
      - ./internal_users.yml:/usr/share/wazuh-indexer/config/opensearch-security/internal_users.yml
```
create a opensearch.yml file with the following content:
```
network.host: 0.0.0.0
network.publish_host: replace_with_your_indexer5_ip
node.name: wazuh5.indexer
node.roles:
  - cluster_manager
  - data
node.attr.temp: warm
cluster.name: "wazuh-cluster"
discovery.seed_hosts:
       - replace_with_your_indexer1_ip
       - replace_with_your_indexer2_ip
       - replace_with_your_indexer3_ip
       - replace_with_your_indexer4_ip
       - replace_with_your_indexer5_ip
path.data: /var/lib/wazuh-indexer
path.logs: /var/log/wazuh-indexer
plugins.security.ssl.http.pemcert_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh5.indexer.pem
plugins.security.ssl.http.pemkey_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh5.indexer.key
plugins.security.ssl.http.pemtrustedcas_filepath: ${OPENSEARCH_PATH_CONF}/certs/root-ca.pem
plugins.security.ssl.transport.pemcert_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh5.indexer.pem
plugins.security.ssl.transport.pemkey_filepath: ${OPENSEARCH_PATH_CONF}/certs/wazuh5.indexer.key
plugins.security.ssl.transport.pemtrustedcas_filepath: ${OPENSEARCH_PATH_CONF}/certs/root-ca.pem
plugins.security.ssl.http.enabled: true
plugins.security.ssl.transport.enforce_hostname_verification: true
plugins.security.ssl.transport.resolve_hostname: false
plugins.security.ssl.http.enabled_ciphers:
 - "TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256"
 - "TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384"
 - "TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256"
 - "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384"
plugins.security.ssl.http.enabled_protocols:
 - "TLSv1.2"
plugins.security.authcz.admin_dn:
 - "CN=admin,OU=SOC,O=CISO,L=Scottsdale,C=US"
plugins.security.check_snapshot_restore_write_privileges: true
plugins.security.enable_snapshot_restore_privilege: true
plugins.security.nodes_dn:
 - "CN=wazuh1.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh2.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh3.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh4.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=wazuh5.indexer,OU=SOC,O=CISO,L=Scottsdale,C=US"
 - "CN=filebeat,OU=SOC,O=CISO,L=Scottsdale,C=US"
plugins.security.restapi.roles_enabled:
 - "all_access"
 - "security_rest_api_access"
plugins.security.allow_default_init_securityindex: true
cluster.routing.allocation.disk.threshold_enabled: false
compatibility.override_main_response_version: true
```
create a internal_users.yml file with the following content:
```
---
# This is the internal user database
# The hash value is a bcrypt hash and can be generated with plugin/tools/hash.sh

_meta:
  type: "internalusers"
  config_version: 2

# Define your internal users here

## Demo users

admin:
  hash: "$2y$12$XiQR9Ju.ppzxljOh9bIrYuwv7R2FI263Vw/kZaqPMum56iIn5KYra"
  reserved: true
  backend_roles:
  - "admin"
  description: "Demo admin user"

kibanaserver:
  hash: "$2a$12$4AcgAt3xwOWadA5s5blL6ev39OXDNhmOesEoo33eZtrq2N0YrU3H."
  reserved: true
  description: "Demo kibanaserver user"

kibanaro:
  hash: "$2a$12$JJSXNfTowz7Uu5ttXfeYpeYE0arACvcwlPBStB1F.MI7f0U9Z4DGC"
  reserved: false
  backend_roles:
  - "kibanauser"
  - "readall"
  attributes:
    attribute1: "value1"
    attribute2: "value2"
    attribute3: "value3"
  description: "Demo kibanaro user"

logstash:
  hash: "$2a$12$u1ShR4l4uBS3Uv59Pa2y5.1uQuZBrZtmNfqB3iM/.jL0XoV9sghS2"
  reserved: false
  backend_roles:
  - "logstash"
  description: "Demo logstash user"

readall:
  hash: "$2a$12$ae4ycwzwvLtZxwZ82RmiEunBbIPiAmGZduBAjKN0TXdwQFtCwARz2"
  reserved: false
  backend_roles:
  - "readall"
  description: "Demo readall user"

snapshotrestore:
  hash: "$2y$12$DpwmetHKwgYnorbgdvORCenv4NAK8cPUg8AI6pxLCuWf/ALc0.v7W"
  reserved: false
  backend_roles:
  - "snapshotrestore"
  description: "Demo snapshotrestore user"
```
Create a .env file with the following content and replace the values with your own:
```
WAZUH_INDEXER5_CONTAINER_NAME=wazuh5.indexer
WAZUH_INDEXER5_HOSTNAME=wazuh5.indexer
WAZUH_INDEXER5_DATA_DIR=/data/Premium/data/
root_ca_pem=./certs/root-ca.pem
wazuh5_indexer_pem=./certs/wazuh5.indexer.pem
wazuh5_indexer_key_pem=./certs/wazuh5.indexer-key.pem
admin_pem=./certs/admin.pem
admin_key_pem=./certs/admin-key.pem
AZURE_ACCOUNT=your_azure_account_name
AZURE_KEY=your_azure_account_key
```
Once you have created the docker-compose.yml, opensearch.yml, internal_users.yml, and .env files, and copied the certificates to the certs directory, you can start the Wazuh indexer5 container by running the following command in the same directory as the docker-compose.yml file:
```
docker-compose up -d
```
check the logs to ensure that the Wazuh indexer5 container is running correctly:
```
docker logs -f wazuh5.indexer
```

### 2.6 Wazuh indexer password change
Once all the Wazuh indexer containers are running, you can change the password for the admin user. To do this, run the following command on any of the indexer nodes:
```
docker exec -it wazuh1.indexer bash

export OPENSEARCH_HOME=/usr/share/wazuh-indexer
export OPENSEARCH_PATH_CONF=/usr/share/wazuh-indexer/config
export OPENSEARCH_JAVA_HOME=/usr/share/wazuh-indexer/jdk

/usr/share/wazuh-indexer/plugins/opensearch-security/tools/hash.sh

You will be prompted to enter a new password for the admin user. Enter the new password and confirm it.
once the password has been changed, copy the new hash value and update the internal_users.yml file on all indexer nodes with the new hash value for the admin user. After updating the internal_users.yml file, restart all the Wazuh indexer containers to apply the changes.
```
Once the container is running, you can move to the next step of deploying Wazuh manager.

## 3. Wazuh manager installation
Please follow the steps below to install Wazuh manager on the corresponding node.

In this guide we are using the path /data/Premium/workspace
Create a docker-compose.yml file with the following content:
```
# Wazuh App Copyright (C) 2017, Wazuh Inc. (License GPLv2)
services:
  wazuh.master:
    image: socacr01.azurecr.io/wazuh-manager:4.14.4
    container_name: ${WAZUH_MASTER_CONTAINER_NAME}
    hostname: ${WAZUH_MASTER_HOSTNAME}
    restart: always
    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 655360
        hard: 655360
    ports:
      - "1514:1514"
      - "1515:1515"
      - "514:514/udp"
      - "55000:55000"
      - "1516:1516"
      - "9065:9065"
    environment:
      - INDEXER_URL="https://${INDEXER_NODE_1}:9200","https://${INDEXER_NODE_2}:9200","https://${INDEXER_NODE_3}:9200","https://${INDEXER_NODE_4}:9200","https://${INDEXER_NODE_5}:9200"
      - INDEXER_USERNAME=${INDEXER_USERNAME}
      - INDEXER_PASSWORD=${INDEXER_PASSWORD}
      - FILEBEAT_SSL_VERIFICATION_MODE=full
      - SSL_CERTIFICATE_AUTHORITIES=/etc/ssl/root-ca.pem
      - SSL_CERTIFICATE=/etc/ssl/filebeat.pem
      - SSL_KEY=/etc/ssl/filebeat.key
      - API_USERNAME=${API_USERNAME}
      - API_PASSWORD=${API_PASSWORD}
    volumes:
      - ${WAZUH_MASTER_DATA_DIR}/master-wazuh-api-configuration:/var/ossec/api/configuration
      - ${WAZUH_MASTER_DATA_DIR}/master-wazuh-etc:/var/ossec/etc
      - ${WAZUH_MASTER_DATA_DIR}/master-wazuh-logs:/var/ossec/logs
      - ${WAZUH_MASTER_DATA_DIR}/master-wazuh-queue:/var/ossec/queue
      - ${WAZUH_MASTER_DATA_DIR}/master-wazuh-var-multigroups:/var/ossec/var/multigroups
      - ${WAZUH_MASTER_DATA_DIR}/master-wazuh-integrations:/var/ossec/integrations
      - ${WAZUH_MASTER_DATA_DIR}/master-wazuh-active-response:/var/ossec/active-response/bin
      - ${WAZUH_MASTER_DATA_DIR}/master-wazuh-agentless:/var/ossec/agentless
      - ${WAZUH_MASTER_DATA_DIR}/master-wazuh-wodles:/var/ossec/wodles
      - ${WAZUH_MASTER_DATA_DIR}/master-filebeat-etc:/etc/filebeat
      - ${WAZUH_MASTER_DATA_DIR}/master-filebeat-var:/var/lib/filebeat
      - ${root_ca_manager}:/etc/ssl/root-ca.pem
      - ${wazuh_master_pem}:/etc/ssl/filebeat.pem
      - ${wazuh_master_key}:/etc/ssl/filebeat.key
      - ./ossec.conf:/wazuh-config-mount/etc/ossec.conf
```
create a ossec.conf file with the following content:
```
<ossec_config>
  <global>
    <jsonout_output>yes</jsonout_output>
    <alerts_log>yes</alerts_log>
    <logall>yes</logall>
    <logall_json>yes</logall_json>
    <email_notification>no</email_notification>
    <smtp_server>smtp.example.wazuh.com</smtp_server>
    <email_from>wazuh@example.wazuh.com</email_from>
    <email_to>recipient@example.wazuh.com</email_to>
    <email_maxperhour>12</email_maxperhour>
    <email_log_source>alerts.log</email_log_source>
    <agents_disconnection_time>10m</agents_disconnection_time>
    <agents_disconnection_alert_time>0</agents_disconnection_alert_time>
  </global>

  <alerts>
    <log_alert_level>3</log_alert_level>
    <email_alert_level>12</email_alert_level>
  </alerts>

  <!-- Choose between "plain", "json", or "plain,json" for the format of internal logs -->
  <logging>
    <log_format>plain</log_format>
  </logging>

  <remote>
    <connection>secure</connection>
    <port>1514</port>
    <protocol>tcp</protocol>
    <queue_size>131072</queue_size>
  </remote>

  <remote>
    <connection>syslog</connection>
    <port>9065</port>
    <protocol>tcp</protocol>
    <allowed-ips>172.18.0.1</allowed-ips>
    <allowed-ips>HAProxy_IP_Address</allowed-ips>
  </remote>

  <!-- Policy monitoring -->
  <rootcheck>
    <disabled>no</disabled>
    <check_files>yes</check_files>
    <check_trojans>yes</check_trojans>
    <check_dev>yes</check_dev>
    <check_sys>yes</check_sys>
    <check_pids>yes</check_pids>
    <check_ports>yes</check_ports>
    <check_if>yes</check_if>

    <!-- Frequency that rootcheck is executed - every 12 hours -->
    <frequency>43200</frequency>

    <rootkit_files>etc/rootcheck/rootkit_files.txt</rootkit_files>
    <rootkit_trojans>etc/rootcheck/rootkit_trojans.txt</rootkit_trojans>

    <skip_nfs>yes</skip_nfs>
  </rootcheck>

  <wodle name="cis-cat">
    <disabled>yes</disabled>
    <timeout>1800</timeout>
    <interval>1d</interval>
    <scan-on-start>yes</scan-on-start>

    <java_path>wodles/java</java_path>
    <ciscat_path>wodles/ciscat</ciscat_path>
  </wodle>

  <!-- Osquery integration -->
  <wodle name="osquery">
    <disabled>yes</disabled>
    <run_daemon>yes</run_daemon>
    <log_path>/var/log/osquery/osqueryd.results.log</log_path>
    <config_path>/etc/osquery/osquery.conf</config_path>
    <add_labels>yes</add_labels>
  </wodle>

  <!-- System inventory -->
  <wodle name="syscollector">
    <disabled>no</disabled>
    <interval>1h</interval>
    <scan_on_start>yes</scan_on_start>
    <hardware>yes</hardware>
    <os>yes</os>
    <network>yes</network>
    <packages>yes</packages>
    <ports all="yes">yes</ports>
    <processes>yes</processes>

    <!-- Database synchronization settings -->
    <synchronization>
      <max_eps>10</max_eps>
    </synchronization>
  </wodle>

  <sca>
    <enabled>yes</enabled>
    <scan_on_start>yes</scan_on_start>
    <interval>12h</interval>
    <skip_nfs>yes</skip_nfs>
  </sca>

  <vulnerability-detection>
    <enabled>yes</enabled>
    <index-status>yes</index-status>
    <feed-update-interval>60m</feed-update-interval>
  </vulnerability-detection>

  <indexer>
    <enabled>yes</enabled>
    <hosts>
      <host>https://indexer1_IP_address:9200</host>
      <host>https://indexer2_IP_address:9200</host>
      <host>https://indexer3_IP_address:9200</host>
    </hosts>
    <ssl>
      <certificate_authorities>
        <ca>/etc/ssl/root-ca.pem</ca>
      </certificate_authorities>
      <certificate>/etc/ssl/filebeat.pem</certificate>
      <key>/etc/ssl/filebeat.key</key>
    </ssl>
  </indexer>

  <!-- File integrity monitoring -->
  <syscheck>
    <disabled>no</disabled>

    <!-- Frequency that syscheck is executed default every 12 hours -->
    <frequency>43200</frequency>

    <scan_on_start>yes</scan_on_start>

    <!-- Generate alert when new file detected -->
    <alert_new_files>yes</alert_new_files>

    <!-- Don't ignore files that change more than 'frequency' times -->
    <auto_ignore frequency="10" timeframe="3600">no</auto_ignore>

    <!-- Directories to check  (perform all possible verifications) -->
    <directories>/etc,/usr/bin,/usr/sbin</directories>
    <directories>/bin,/sbin,/boot</directories>

    <!-- Files/directories to ignore -->
    <ignore>/etc/mtab</ignore>
    <ignore>/etc/hosts.deny</ignore>
    <ignore>/etc/mail/statistics</ignore>
    <ignore>/etc/random-seed</ignore>
    <ignore>/etc/random.seed</ignore>
    <ignore>/etc/adjtime</ignore>
    <ignore>/etc/httpd/logs</ignore>
    <ignore>/etc/utmpx</ignore>
    <ignore>/etc/wtmpx</ignore>
    <ignore>/etc/cups/certs</ignore>
    <ignore>/etc/dumpdates</ignore>
    <ignore>/etc/svc/volatile</ignore>

    <!-- File types to ignore -->
    <ignore type="sregex">.log$|.swp$</ignore>

    <!-- Check the file, but never compute the diff -->
    <nodiff>/etc/ssl/private.key</nodiff>

    <skip_nfs>yes</skip_nfs>
    <skip_dev>yes</skip_dev>
    <skip_proc>yes</skip_proc>
    <skip_sys>yes</skip_sys>

    <!-- Nice value for Syscheck process -->
    <process_priority>10</process_priority>

    <!-- Maximum output throughput -->
    <max_eps>100</max_eps>

    <!-- Database synchronization settings -->
    <synchronization>
      <enabled>yes</enabled>
      <interval>5m</interval>
      <max_interval>1h</max_interval>
      <max_eps>10</max_eps>
    </synchronization>
  </syscheck>

  <!-- Active response -->
  <global>
    <white_list>127.0.0.1</white_list>
    <white_list>^localhost.localdomain$</white_list>
  </global>

  <command>
    <name>disable-account</name>
    <executable>disable-account</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>restart-wazuh</name>
    <executable>restart-wazuh</executable>
  </command>

  <command>
    <name>firewall-drop</name>
    <executable>firewall-drop</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>host-deny</name>
    <executable>host-deny</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>route-null</name>
    <executable>route-null</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>win_route-null</name>
    <executable>route-null.exe</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>netsh</name>
    <executable>netsh.exe</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <!--
  <active-response>
    active-response options here
  </active-response>
  -->

  <!-- Log analysis -->
  <localfile>
    <log_format>command</log_format>
    <command>df -P</command>
    <frequency>360</frequency>
  </localfile>

  <localfile>
    <log_format>full_command</log_format>
    <command>netstat -tulpn | sed 's/\([[:alnum:]]\+\)\ \+[[:digit:]]\+\ \+[[:digit:]]\+\ \+\(.*\):\([[:digit:]]*\)\ \+\([0-9\.\:\*]\+\).\+\ \([[:digit:]]*\/[[:alnum:]\-]*\).*/\1 \2 == \3 == \4 \5/' | sort -k 4 -g | sed 's/ == \(.*\) ==/:\1/' | sed 1,2d</command>
    <alias>netstat listening ports</alias>
    <frequency>360</frequency>
  </localfile>

  <localfile>
    <log_format>full_command</log_format>
    <command>last -n 20</command>
    <frequency>360</frequency>
  </localfile>

  <ruleset>
    <!-- Default ruleset -->
    <decoder_dir>ruleset/decoders</decoder_dir>
    <rule_dir>ruleset/rules</rule_dir>
    <rule_exclude>0215-policy_rules.xml</rule_exclude>
    <list>etc/lists/audit-keys</list>
    <list>etc/lists/amazon/aws-eventnames</list>
    <list>etc/lists/security-eventchannel</list>
    <list>etc/lists/malicious-ioc/malicious-ip</list>
    <list>etc/lists/malicious-ioc/malicious-domains</list>
    <list>etc/lists/malicious-ioc/malware-hashes</list>

    <!-- User-defined ruleset -->
    <decoder_dir>etc/decoders</decoder_dir>
    <rule_dir>etc/rules</rule_dir>
  </ruleset>

  <rule_test>
    <enabled>yes</enabled>
    <threads>1</threads>
    <max_sessions>64</max_sessions>
    <session_timeout>15m</session_timeout>
  </rule_test>

  <!-- Configuration for wazuh-authd -->
  <auth>
    <disabled>no</disabled>
    <port>1515</port>
    <use_source_ip>no</use_source_ip>
    <purge>yes</purge>
    <use_password>no</use_password>
    <ciphers>HIGH:!ADH:!EXP:!MD5:!RC4:!3DES:!CAMELLIA:@STRENGTH</ciphers>
    <!-- <ssl_agent_ca></ssl_agent_ca> -->
    <ssl_verify_host>no</ssl_verify_host>
    <ssl_manager_cert>etc/sslmanager.cert</ssl_manager_cert>
    <ssl_manager_key>etc/sslmanager.key</ssl_manager_key>
    <ssl_auto_negotiate>no</ssl_auto_negotiate>
  </auth>

  <cluster>
    <name>wazuh</name>
    <node_name>manager</node_name>
    <node_type>master</node_type>
    <key>c98b6ha9b6169zc5f67rae55ae4z5647</key>
    <port>1516</port>
    <bind_addr>0.0.0.0</bind_addr>
    <nodes>
        <node>wazuh.master</node>
    </nodes>
    <hidden>no</hidden>
    <disabled>no</disabled>
  </cluster>

</ossec_config>

<ossec_config>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/ossec/logs/active-responses.log</location>
  </localfile>

</ossec_config>
```
replace the following placeholders in the ossec.conf file with your own values:
- replace `indexer1_IP_address`, `indexer2_IP_address`, and `indexer3_IP_address` with the IP addresses of your Wazuh indexer nodes.
- replace `HAProxy_IP_Address` with the IP address of your HAProxy server.  

create a .env file with the following content and replace the values with your own:
```
WAZUH_MASTER_CONTAINER_NAME=wazuh.master
WAZUH_MASTER_HOSTNAME=wazuh.master
WAZUH_MASTER_DATA_DIR=/data/Premium/data/
INDEXER_NODE_1=indexer1_IP_address
INDEXER_NODE_2=indexer2_IP_address
INDEXER_NODE_3=indexer3_IP_address
INDEXER_NODE_4=indexer4_IP_address
INDEXER_NODE_5=indexer5_IP_address
INDEXER_USERNAME=admin
INDEXER_PASSWORD=your_indexer_password
API_USERNAME=wazuh-wui
API_PASSWORD=your_wazuh_api_password
root_ca_manager=./certs/root-ca.pem
wazuh_master_pem=./certs/wazuh.master.pem
wazuh_master_key=./certs/wazuh.master-key.pem
```
replace the following placeholders in the .env file with your own values:
- replace `indexer1_IP_address`, `indexer2_IP_address`, `indexer3_IP_address`, `indexer4_IP_address`, and `indexer5_IP_address` with the IP addresses of your Wazuh indexer nodes.
- replace `your_indexer_password` with the password for the Wazuh indexer.  
- replace `your_wazuh_api_password` with the password for the Wazuh API.

Once you have created the docker-compose.yml, ossec.conf, and .env files, and copied the certificates to the certs directory, you can start the Wazuh manager container by running the following command in the same directory as the docker-compose.yml file:
``` 
docker-compose up -d
```
check the logs to ensure that the Wazuh manager container is running correctly:
```
docker logs -f wazuh.master 
```
Once the container is running, you can move to the next step of deploying Wazuh worker nodes.

## 4. Wazuh worker nodes installation
Please follow the steps below to install Wazuh worker nodes on the corresponding nodes.

### 4.1 Wazuh worker1 installation
In this guide we are using the path /data/Premium/workspace
Create a docker-compose.yml file with the following content:
```
# Wazuh App Copyright (C) 2017, Wazuh Inc. (License GPLv2)
services:
  wazuh.worker:
    image: socacr01.azurecr.io/wazuh-manager:4.14.4
    container_name: ${WAZUH_WORKER1_CONTAINER_NAME}
    hostname: ${WAZUH_WORKER1_HOSTNAME}
    restart: always
    ports:
      - "1514:1514"
      - "1515:1515"
      - "514:514/udp"
      - "55000:55000"
      - "1516:1516"
      - "9065:9065"

    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 655360
        hard: 655360
    environment:
      - INDEXER_URL="https://${INDEXER_NODE_1}:9200","https://${INDEXER_NODE_2}:9200","https://${INDEXER_NODE_3}:9200","https://${INDEXER_NODE_4}:9200","https://${INDEXER_NODE_5}:9200"
      - INDEXER_USERNAME=${INDEXER_USERNAME}
      - INDEXER_PASSWORD=${INDEXER_PASSWORD}
      - FILEBEAT_SSL_VERIFICATION_MODE=full
      - SSL_CERTIFICATE_AUTHORITIES=/etc/ssl/root-ca.pem
      - SSL_CERTIFICATE=/etc/ssl/filebeat.pem
      - SSL_KEY=/etc/ssl/filebeat.key
    volumes:
      - ${WAZUH_WORKER1_DATA_DIR}/worker-wazuh-api-configuration:/var/ossec/api/configuration
      - ${WAZUH_WORKER1_DATA_DIR}/worker-wazuh-etc:/var/ossec/etc
      - ${WAZUH_WORKER1_DATA_DIR}/worker-wazuh-logs:/var/ossec/logs
      - ${WAZUH_WORKER1_DATA_DIR}/worker-wazuh-queue:/var/ossec/queue
      - ${WAZUH_WORKER1_DATA_DIR}/worker-wazuh-var-multigroups:/var/ossec/var/multigroups
      - ${WAZUH_WORKER1_DATA_DIR}/worker-wazuh-integrations:/var/ossec/integrations
      - ${WAZUH_WORKER1_DATA_DIR}/worker-wazuh-active-response:/var/ossec/active-response/bin
      - ${WAZUH_WORKER1_DATA_DIR}/worker-wazuh-agentless:/var/ossec/agentless
      - ${WAZUH_WORKER1_DATA_DIR}/worker-wazuh-wodles:/var/ossec/wodles
      - ${WAZUH_WORKER1_DATA_DIR}/worker-filebeat-etc:/etc/filebeat
      - ${WAZUH_WORKER1_DATA_DIR}/worker-filebeat-var:/var/lib/filebeat
      - ${root_ca_manager}:/etc/ssl/root-ca.pem
      - ${wazuh_worker1_pem}:/etc/ssl/filebeat.pem
      - ${wazuh_worker1_key}:/etc/ssl/filebeat.key
      - ./ossec.conf:/wazuh-config-mount/etc/ossec.conf
```
create a ossec.conf file with the following content:
```
<ossec_config>
  <global>
    <jsonout_output>yes</jsonout_output>
    <alerts_log>yes</alerts_log>
    <logall>yes</logall>
    <logall_json>yes</logall_json>
    <email_notification>no</email_notification>
    <smtp_server>smtp.example.wazuh.com</smtp_server>
    <email_from>wazuh@example.wazuh.com</email_from>
    <email_to>recipient@example.wazuh.com</email_to>
    <email_maxperhour>12</email_maxperhour>
    <email_log_source>alerts.log</email_log_source>
    <agents_disconnection_time>10m</agents_disconnection_time>
    <agents_disconnection_alert_time>0</agents_disconnection_alert_time>
  </global>

  <alerts>
    <log_alert_level>3</log_alert_level>
    <email_alert_level>12</email_alert_level>
  </alerts>

  <!-- Choose between "plain", "json", or "plain,json" for the format of internal logs -->
  <logging>
    <log_format>plain</log_format>
  </logging>

  <remote>
    <connection>secure</connection>
    <port>1514</port>
    <protocol>tcp</protocol>
    <queue_size>131072</queue_size>
  </remote>

  <remote>
    <connection>syslog</connection>
    <port>9065</port>
    <protocol>tcp</protocol>
    <allowed-ips>172.18.0.1</allowed-ips>
    <allowed-ips>HAProxy_IP_address</allowed-ips>
  </remote>

  <!-- Policy monitoring -->
  <rootcheck>
    <disabled>no</disabled>
    <check_files>yes</check_files>
    <check_trojans>yes</check_trojans>
    <check_dev>yes</check_dev>
    <check_sys>yes</check_sys>
    <check_pids>yes</check_pids>
    <check_ports>yes</check_ports>
    <check_if>yes</check_if>

    <!-- Frequency that rootcheck is executed - every 12 hours -->
    <frequency>43200</frequency>

    <rootkit_files>etc/rootcheck/rootkit_files.txt</rootkit_files>
    <rootkit_trojans>etc/rootcheck/rootkit_trojans.txt</rootkit_trojans>

    <skip_nfs>yes</skip_nfs>
  </rootcheck>

  <wodle name="cis-cat">
    <disabled>yes</disabled>
    <timeout>1800</timeout>
    <interval>1d</interval>
    <scan-on-start>yes</scan-on-start>

    <java_path>wodles/java</java_path>
    <ciscat_path>wodles/ciscat</ciscat_path>
  </wodle>

  <!-- Osquery integration -->
  <wodle name="osquery">
    <disabled>yes</disabled>
    <run_daemon>yes</run_daemon>
    <log_path>/var/log/osquery/osqueryd.results.log</log_path>
    <config_path>/etc/osquery/osquery.conf</config_path>
    <add_labels>yes</add_labels>
  </wodle>

  <!-- System inventory -->
  <wodle name="syscollector">
    <disabled>no</disabled>
    <interval>1h</interval>
    <scan_on_start>yes</scan_on_start>
    <hardware>yes</hardware>
    <os>yes</os>
    <network>yes</network>
    <packages>yes</packages>
    <ports all="yes">yes</ports>
    <processes>yes</processes>

    <!-- Database synchronization settings -->
    <synchronization>
      <max_eps>10</max_eps>
    </synchronization>
  </wodle>

  <sca>
    <enabled>yes</enabled>
    <scan_on_start>yes</scan_on_start>
    <interval>12h</interval>
    <skip_nfs>yes</skip_nfs>
  </sca>

  <vulnerability-detection>
    <enabled>yes</enabled>
    <index-status>yes</index-status>
    <feed-update-interval>60m</feed-update-interval>
  </vulnerability-detection>

  <indexer>
    <enabled>yes</enabled>
    <hosts>
      <host>https://indexer1_IP_address:9200</host>
      <host>https://indexer2_IP_address:9200</host>
      <host>https://indexer3_IP_address:9200</host>
    </hosts>
    <ssl>
      <certificate_authorities>
        <ca>/etc/ssl/root-ca.pem</ca>
      </certificate_authorities>
      <certificate>/etc/ssl/filebeat.pem</certificate>
      <key>/etc/ssl/filebeat.key</key>
    </ssl>
  </indexer>

  <!-- File integrity monitoring -->
  <syscheck>
    <disabled>no</disabled>

    <!-- Frequency that syscheck is executed default every 12 hours -->
    <frequency>43200</frequency>

    <scan_on_start>yes</scan_on_start>

    <!-- Generate alert when new file detected -->
    <alert_new_files>yes</alert_new_files>

    <!-- Don't ignore files that change more than 'frequency' times -->
    <auto_ignore frequency="10" timeframe="3600">no</auto_ignore>

    <!-- Directories to check  (perform all possible verifications) -->
    <directories>/etc,/usr/bin,/usr/sbin</directories>
    <directories>/bin,/sbin,/boot</directories>

    <!-- Files/directories to ignore -->
    <ignore>/etc/mtab</ignore>
    <ignore>/etc/hosts.deny</ignore>
    <ignore>/etc/mail/statistics</ignore>
    <ignore>/etc/random-seed</ignore>
    <ignore>/etc/random.seed</ignore>
    <ignore>/etc/adjtime</ignore>
    <ignore>/etc/httpd/logs</ignore>
    <ignore>/etc/utmpx</ignore>
    <ignore>/etc/wtmpx</ignore>
    <ignore>/etc/cups/certs</ignore>
    <ignore>/etc/dumpdates</ignore>
    <ignore>/etc/svc/volatile</ignore>

    <!-- File types to ignore -->
    <ignore type="sregex">.log$|.swp$</ignore>

    <!-- Check the file, but never compute the diff -->
    <nodiff>/etc/ssl/private.key</nodiff>

    <skip_nfs>yes</skip_nfs>
    <skip_dev>yes</skip_dev>
    <skip_proc>yes</skip_proc>
    <skip_sys>yes</skip_sys>

    <!-- Nice value for Syscheck process -->
    <process_priority>10</process_priority>

    <!-- Maximum output throughput -->
    <max_eps>100</max_eps>

    <!-- Database synchronization settings -->
    <synchronization>
      <enabled>yes</enabled>
      <interval>5m</interval>
      <max_interval>1h</max_interval>
      <max_eps>10</max_eps>
    </synchronization>
  </syscheck>

  <!-- Active response -->
  <global>
    <white_list>127.0.0.1</white_list>
    <white_list>^localhost.localdomain$</white_list>
  </global>

  <command>
    <name>disable-account</name>
    <executable>disable-account</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>restart-wazuh</name>
    <executable>restart-wazuh</executable>
  </command>

  <command>
    <name>firewall-drop</name>
    <executable>firewall-drop</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>host-deny</name>
    <executable>host-deny</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>route-null</name>
    <executable>route-null</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>win_route-null</name>
    <executable>route-null.exe</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>netsh</name>
    <executable>netsh.exe</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <!--
  <active-response>
    active-response options here
  </active-response>
  -->

  <!-- Log analysis -->
  <localfile>
    <log_format>command</log_format>
    <command>df -P</command>
    <frequency>360</frequency>
  </localfile>

  <localfile>
    <log_format>full_command</log_format>
    <command>netstat -tulpn | sed 's/\([[:alnum:]]\+\)\ \+[[:digit:]]\+\ \+[[:digit:]]\+\ \+\(.*\):\([[:digit:]]*\)\ \+\([0-9\.\:\*]\+\).\+\ \([[:digit:]]*\/[[:alnum:]\-]*\).*/\1 \2 == \3 == \4 \5/' | sort -k 4 -g | sed 's/ == \(.*\) ==/:\1/' | sed 1,2d</command>
    <alias>netstat listening ports</alias>
    <frequency>360</frequency>
  </localfile>

  <localfile>
    <log_format>full_command</log_format>
    <command>last -n 20</command>
    <frequency>360</frequency>
  </localfile>

  <ruleset>
    <!-- Default ruleset -->
    <decoder_dir>ruleset/decoders</decoder_dir>
    <rule_dir>ruleset/rules</rule_dir>
    <rule_exclude>0215-policy_rules.xml</rule_exclude>
    <list>etc/lists/audit-keys</list>
    <list>etc/lists/amazon/aws-eventnames</list>
    <list>etc/lists/security-eventchannel</list>
    <list>etc/lists/malicious-ioc/malicious-ip</list>
    <list>etc/lists/malicious-ioc/malicious-domains</list>
    <list>etc/lists/malicious-ioc/malware-hashes</list>

    <!-- User-defined ruleset -->
    <decoder_dir>etc/decoders</decoder_dir>
    <rule_dir>etc/rules</rule_dir>
  </ruleset>

  <rule_test>
    <enabled>yes</enabled>
    <threads>1</threads>
    <max_sessions>64</max_sessions>
    <session_timeout>15m</session_timeout>
  </rule_test>

  <!-- Configuration for wazuh-authd -->
  <auth>
    <disabled>no</disabled>
    <port>1515</port>
    <use_source_ip>no</use_source_ip>
    <purge>yes</purge>
    <use_password>no</use_password>
    <ciphers>HIGH:!ADH:!EXP:!MD5:!RC4:!3DES:!CAMELLIA:@STRENGTH</ciphers>
    <!-- <ssl_agent_ca></ssl_agent_ca> -->
    <ssl_verify_host>no</ssl_verify_host>
    <ssl_manager_cert>etc/sslmanager.cert</ssl_manager_cert>
    <ssl_manager_key>etc/sslmanager.key</ssl_manager_key>
    <ssl_auto_negotiate>no</ssl_auto_negotiate>
  </auth>

  <cluster>
    <name>wazuh</name>
    <node_name>worker01</node_name>
    <node_type>worker</node_type>
    <key>c98b6ha9b6169zc5f67rae55ae4z5647</key>
    <port>1516</port>
    <bind_addr>0.0.0.0</bind_addr>
    <nodes>
        <node>manager_IP_address</node>
    </nodes>
    <hidden>no</hidden>
    <disabled>no</disabled>
  </cluster>

</ossec_config>

<ossec_config>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/ossec/logs/active-responses.log</location>
  </localfile>

</ossec_config>
```
Replace the following placeholders in the ossec.conf file with your own values:
- replace `indexer1_IP_address`, `indexer2_IP_address`, and `indexer3_IP_address` with the IP addresses of your Wazuh indexer nodes.
- replace `HAProxy_IP_address` with the IP address of your HAProxy server.
- replace `manager_IP_address` with the IP address of your Wazuh manager node.

create a .env file with the following content and replace the values with your own:
```
WAZUH_WORKER1_CONTAINER_NAME=wazuh.worker1
WAZUH_WORKER1_HOSTNAME=wazuh.worker
WAZUH_WORKER1_DATA_DIR=/data/Premium/data
INDEXER_NODE_1=indexer1_IP_address
INDEXER_NODE_2=indexer2_IP_address
INDEXER_NODE_3=indexer3_IP_address
INDEXER_NODE_4=indexer4_IP_address
INDEXER_NODE_5=indexer5_IP_address
INDEXER_USERNAME=admin
INDEXER_PASSWORD=your_indexer_password
root_ca_manager=./certs/root-ca.pem
wazuh_worker1_pem=./certs/wazuh.worker1.pem
wazuh_worker1_key=./certs/wazuh.worker1-key.pem
```
Replace the following placeholders in the .env file with your own values:
- replace `indexer1_IP_address`, `indexer2_IP_address`, `indexer3_IP_address`, `indexer4_IP_address`, and `indexer5_IP_address` with the IP addresses of your Wazuh indexer nodes.
- replace `your_indexer_password` with the password for the Wazuh indexer.  
Once you have created the docker-compose.yml, ossec.conf, and .env files, and copied the certificates to the certs directory, you can start the Wazuh worker1 container by running the following command in the same directory as the docker-compose.yml file:
```
docker-compose up -d
```
Check the logs to ensure that the Wazuh worker1 container is running correctly:
```
docker logs -f wazuh.worker1
```
Once the container is running, you can move to the next step of deploying Wazuh worker2 nodes.

### 4.2 Wazuh worker2 installation
In this guide we are using the path /data/Premium/workspace
Create a docker-compose.yml file with the following content:
```
# Wazuh App Copyright (C) 2017, Wazuh Inc. (License GPLv2)
services:
  wazuh.worker:
    image: socacr01.azurecr.io/wazuh-manager:4.14.4
    container_name: ${WAZUH_WORKER2_CONTAINER_NAME}
    hostname: ${WAZUH_WORKER2_HOSTNAME}
    restart: always
    ports:
      - "1514:1514"
      - "1515:1515"
      - "514:514/udp"
      - "55000:55000"
      - "1516:1516"
      - "9065:9065"

    ulimits:
      memlock:
        soft: -1
        hard: -1
      nofile:
        soft: 655360
        hard: 655360
    environment:
      - INDEXER_URL="https://${INDEXER_NODE_1}:9200","https://${INDEXER_NODE_2}:9200","https://${INDEXER_NODE_3}:9200","https://${INDEXER_NODE_4}:9200","https://${INDEXER_NODE_5}:9200"
      - INDEXER_USERNAME=${INDEXER_USERNAME}
      - INDEXER_PASSWORD=${INDEXER_PASSWORD}
      - FILEBEAT_SSL_VERIFICATION_MODE=full
      - SSL_CERTIFICATE_AUTHORITIES=/etc/ssl/root-ca.pem
      - SSL_CERTIFICATE=/etc/ssl/filebeat.pem
      - SSL_KEY=/etc/ssl/filebeat.key
    volumes:
      - ${WAZUH_WORKER2_DATA_DIR}/worker-wazuh-api-configuration:/var/ossec/api/configuration
      - ${WAZUH_WORKER2_DATA_DIR}/worker-wazuh-etc:/var/ossec/etc
      - ${WAZUH_WORKER2_DATA_DIR}/worker-wazuh-logs:/var/ossec/logs
      - ${WAZUH_WORKER2_DATA_DIR}/worker-wazuh-queue:/var/ossec/queue
      - ${WAZUH_WORKER2_DATA_DIR}/worker-wazuh-var-multigroups:/var/ossec/var/multigroups
      - ${WAZUH_WORKER2_DATA_DIR}/worker-wazuh-integrations:/var/ossec/integrations
      - ${WAZUH_WORKER2_DATA_DIR}/worker-wazuh-active-response:/var/ossec/active-response/bin
      - ${WAZUH_WORKER2_DATA_DIR}/worker-wazuh-agentless:/var/ossec/agentless
      - ${WAZUH_WORKER2_DATA_DIR}/worker-wazuh-wodles:/var/ossec/wodles
      - ${WAZUH_WORKER2_DATA_DIR}/worker-filebeat-etc:/etc/filebeat
      - ${WAZUH_WORKER2_DATA_DIR}/worker-filebeat-var:/var/lib/filebeat
      - ${root_ca_manager}:/etc/ssl/root-ca.pem
      - ${wazuh_worker2_pem}:/etc/ssl/filebeat.pem
      - ${wazuh_worker2_key}:/etc/ssl/filebeat.key
      - ./ossec.conf:/wazuh-config-mount/etc/ossec.conf
```
Create a ossec.conf file with the following content:
```
<ossec_config>
  <global>
    <jsonout_output>yes</jsonout_output>
    <alerts_log>yes</alerts_log>
    <logall>yes</logall>
    <logall_json>yes</logall_json>
    <email_notification>no</email_notification>
    <smtp_server>smtp.example.wazuh.com</smtp_server>
    <email_from>wazuh@example.wazuh.com</email_from>
    <email_to>recipient@example.wazuh.com</email_to>
    <email_maxperhour>12</email_maxperhour>
    <email_log_source>alerts.log</email_log_source>
    <agents_disconnection_time>10m</agents_disconnection_time>
    <agents_disconnection_alert_time>0</agents_disconnection_alert_time>
  </global>

  <alerts>
    <log_alert_level>3</log_alert_level>
    <email_alert_level>12</email_alert_level>
  </alerts>

  <!-- Choose between "plain", "json", or "plain,json" for the format of internal logs -->
  <logging>
    <log_format>plain</log_format>
  </logging>

  <remote>
    <connection>secure</connection>
    <port>1514</port>
    <protocol>tcp</protocol>
    <queue_size>131072</queue_size>
  </remote>

  <remote>
    <connection>syslog</connection>
    <port>9065</port>
    <protocol>tcp</protocol>
    <allowed-ips>172.18.0.1</allowed-ips>
    <allowed-ips>HAProxy_IP_address</allowed-ips>
  </remote>

  <!-- Policy monitoring -->
  <rootcheck>
    <disabled>no</disabled>
    <check_files>yes</check_files>
    <check_trojans>yes</check_trojans>
    <check_dev>yes</check_dev>
    <check_sys>yes</check_sys>
    <check_pids>yes</check_pids>
    <check_ports>yes</check_ports>
    <check_if>yes</check_if>

    <!-- Frequency that rootcheck is executed - every 12 hours -->
    <frequency>43200</frequency>

    <rootkit_files>etc/rootcheck/rootkit_files.txt</rootkit_files>
    <rootkit_trojans>etc/rootcheck/rootkit_trojans.txt</rootkit_trojans>

    <skip_nfs>yes</skip_nfs>
  </rootcheck>

  <wodle name="cis-cat">
    <disabled>yes</disabled>
    <timeout>1800</timeout>
    <interval>1d</interval>
    <scan-on-start>yes</scan-on-start>

    <java_path>wodles/java</java_path>
    <ciscat_path>wodles/ciscat</ciscat_path>
  </wodle>

  <!-- Osquery integration -->
  <wodle name="osquery">
    <disabled>yes</disabled>
    <run_daemon>yes</run_daemon>
    <log_path>/var/log/osquery/osqueryd.results.log</log_path>
    <config_path>/etc/osquery/osquery.conf</config_path>
    <add_labels>yes</add_labels>
  </wodle>

  <!-- System inventory -->
  <wodle name="syscollector">
    <disabled>no</disabled>
    <interval>1h</interval>
    <scan_on_start>yes</scan_on_start>
    <hardware>yes</hardware>
    <os>yes</os>
    <network>yes</network>
    <packages>yes</packages>
    <ports all="yes">yes</ports>
    <processes>yes</processes>

    <!-- Database synchronization settings -->
    <synchronization>
      <max_eps>10</max_eps>
    </synchronization>
  </wodle>

  <sca>
    <enabled>yes</enabled>
    <scan_on_start>yes</scan_on_start>
    <interval>12h</interval>
    <skip_nfs>yes</skip_nfs>
  </sca>

  <vulnerability-detection>
    <enabled>yes</enabled>
    <index-status>yes</index-status>
    <feed-update-interval>60m</feed-update-interval>
  </vulnerability-detection>

  <indexer>
    <enabled>yes</enabled>
    <hosts>
      <host>https://indexer1_IP_address:9200</host>
      <host>https://indexer2_IP_address:9200</host>
      <host>https://indexer3_IP_address:9200</host>
    </hosts>
    <ssl>
      <certificate_authorities>
        <ca>/etc/ssl/root-ca.pem</ca>
      </certificate_authorities>
      <certificate>/etc/ssl/filebeat.pem</certificate>
      <key>/etc/ssl/filebeat.key</key>
    </ssl>
  </indexer>

  <!-- File integrity monitoring -->
  <syscheck>
    <disabled>no</disabled>

    <!-- Frequency that syscheck is executed default every 12 hours -->
    <frequency>43200</frequency>

    <scan_on_start>yes</scan_on_start>

    <!-- Generate alert when new file detected -->
    <alert_new_files>yes</alert_new_files>

    <!-- Don't ignore files that change more than 'frequency' times -->
    <auto_ignore frequency="10" timeframe="3600">no</auto_ignore>

    <!-- Directories to check  (perform all possible verifications) -->
    <directories>/etc,/usr/bin,/usr/sbin</directories>
    <directories>/bin,/sbin,/boot</directories>

    <!-- Files/directories to ignore -->
    <ignore>/etc/mtab</ignore>
    <ignore>/etc/hosts.deny</ignore>
    <ignore>/etc/mail/statistics</ignore>
    <ignore>/etc/random-seed</ignore>
    <ignore>/etc/random.seed</ignore>
    <ignore>/etc/adjtime</ignore>
    <ignore>/etc/httpd/logs</ignore>
    <ignore>/etc/utmpx</ignore>
    <ignore>/etc/wtmpx</ignore>
    <ignore>/etc/cups/certs</ignore>
    <ignore>/etc/dumpdates</ignore>
    <ignore>/etc/svc/volatile</ignore>

    <!-- File types to ignore -->
    <ignore type="sregex">.log$|.swp$</ignore>

    <!-- Check the file, but never compute the diff -->
    <nodiff>/etc/ssl/private.key</nodiff>

    <skip_nfs>yes</skip_nfs>
    <skip_dev>yes</skip_dev>
    <skip_proc>yes</skip_proc>
    <skip_sys>yes</skip_sys>

    <!-- Nice value for Syscheck process -->
    <process_priority>10</process_priority>

    <!-- Maximum output throughput -->
    <max_eps>100</max_eps>

    <!-- Database synchronization settings -->
    <synchronization>
      <enabled>yes</enabled>
      <interval>5m</interval>
      <max_interval>1h</max_interval>
      <max_eps>10</max_eps>
    </synchronization>
  </syscheck>

  <!-- Active response -->
  <global>
    <white_list>127.0.0.1</white_list>
    <white_list>^localhost.localdomain$</white_list>
  </global>

  <command>
    <name>disable-account</name>
    <executable>disable-account</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>restart-wazuh</name>
    <executable>restart-wazuh</executable>
  </command>

  <command>
    <name>firewall-drop</name>
    <executable>firewall-drop</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>host-deny</name>
    <executable>host-deny</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>route-null</name>
    <executable>route-null</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>win_route-null</name>
    <executable>route-null.exe</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <command>
    <name>netsh</name>
    <executable>netsh.exe</executable>
    <timeout_allowed>yes</timeout_allowed>
  </command>

  <!--
  <active-response>
    active-response options here
  </active-response>
  -->

  <!-- Log analysis -->
  <localfile>
    <log_format>command</log_format>
    <command>df -P</command>
    <frequency>360</frequency>
  </localfile>

  <localfile>
    <log_format>full_command</log_format>
    <command>netstat -tulpn | sed 's/\([[:alnum:]]\+\)\ \+[[:digit:]]\+\ \+[[:digit:]]\+\ \+\(.*\):\([[:digit:]]*\)\ \+\([0-9\.\:\*]\+\).\+\ \([[:digit:]]*\/[[:alnum:]\-]*\).*/\1 \2 == \3 == \4 \5/' | sort -k 4 -g | sed 's/ == \(.*\) ==/:\1/' | sed 1,2d</command>
    <alias>netstat listening ports</alias>
    <frequency>360</frequency>
  </localfile>

  <localfile>
    <log_format>full_command</log_format>
    <command>last -n 20</command>
    <frequency>360</frequency>
  </localfile>

  <ruleset>
    <!-- Default ruleset -->
    <decoder_dir>ruleset/decoders</decoder_dir>
    <rule_dir>ruleset/rules</rule_dir>
    <rule_exclude>0215-policy_rules.xml</rule_exclude>
    <list>etc/lists/audit-keys</list>
    <list>etc/lists/amazon/aws-eventnames</list>
    <list>etc/lists/security-eventchannel</list>
    <list>etc/lists/malicious-ioc/malicious-ip</list>
    <list>etc/lists/malicious-ioc/malicious-domains</list>
    <list>etc/lists/malicious-ioc/malware-hashes</list>

    <!-- User-defined ruleset -->
    <decoder_dir>etc/decoders</decoder_dir>
    <rule_dir>etc/rules</rule_dir>
  </ruleset>

  <rule_test>
    <enabled>yes</enabled>
    <threads>1</threads>
    <max_sessions>64</max_sessions>
    <session_timeout>15m</session_timeout>
  </rule_test>

  <!-- Configuration for wazuh-authd -->
  <auth>
    <disabled>no</disabled>
    <port>1515</port>
    <use_source_ip>no</use_source_ip>
    <purge>yes</purge>
    <use_password>no</use_password>
    <ciphers>HIGH:!ADH:!EXP:!MD5:!RC4:!3DES:!CAMELLIA:@STRENGTH</ciphers>
    <!-- <ssl_agent_ca></ssl_agent_ca> -->
    <ssl_verify_host>no</ssl_verify_host>
    <ssl_manager_cert>etc/sslmanager.cert</ssl_manager_cert>
    <ssl_manager_key>etc/sslmanager.key</ssl_manager_key>
    <ssl_auto_negotiate>no</ssl_auto_negotiate>
  </auth>

  <cluster>
    <name>wazuh</name>
    <node_name>worker02</node_name>
    <node_type>worker</node_type>
    <key>c98b6ha9b6169zc5f67rae55ae4z5647</key>
    <port>1516</port>
    <bind_addr>0.0.0.0</bind_addr>
    <nodes>
        <node>manager_IP_address</node>
    </nodes>
    <hidden>no</hidden>
    <disabled>no</disabled>
  </cluster>

</ossec_config>

<ossec_config>
  <localfile>
    <log_format>syslog</log_format>
    <location>/var/ossec/logs/active-responses.log</location>
  </localfile>

</ossec_config>
```
Replace the following placeholders in the ossec.conf file with your own values:
- replace `indexer1_IP_address`, `indexer2_IP_address`, and `indexer3_IP_address` with the IP addresses of your Wazuh indexer nodes.
- replace `HAProxy_IP_address` with the IP address of your HAProxy server.
- replace `manager_IP_address` with the IP address of your Wazuh manager node.

Create a .env file with the following content and replace the values with your own:
```
WAZUH_WORKER2_CONTAINER_NAME=wazuh.worker2
WAZUH_WORKER2_HOSTNAME=wazuh.worker2
WAZUH_WORKER2_DATA_DIR=/data/Premium/data
INDEXER_NODE_1=indexer1_IP_address
INDEXER_NODE_2=indexer2_IP_address
INDEXER_NODE_3=indexer3_IP_address
INDEXER_NODE_4=indexer4_IP_address
INDEXER_NODE_5=indexer5_IP_address
INDEXER_USERNAME=admin
INDEXER_PASSWORD=your_indexer_password
root_ca_manager=./certs/root-ca.pem
wazuh_worker2_pem=./certs/wazuh.worker2.pem
wazuh_worker2_key=./certs/wazuh.worker2-key.pem
```
Replace the following placeholders in the .env file with your own values:
- replace `indexer1_IP_address`, `indexer2_IP_address`, `indexer3_IP_address`, `indexer4_IP_address`, and `indexer5_IP_address` with the IP addresses of your Wazuh indexer nodes.
- replace `your_indexer_password` with the password for the Wazuh indexer.
Once you have created the docker-compose.yml, ossec.conf, and .env files for Wazuh worker2, and copied the certificates to the certs directory, you can start the Wazuh worker2 container by running the following command in the same directory as the docker-compose.yml file:
```
docker-compose up -d
```
Check the logs to ensure that the Wazuh worker2 container is running correctly:
```
docker-compose logs -f wazuh.worker2
```
Once the container is running, you can move to the next step of deploying Wazuh dashboard node.

## 5. Wazuh dashboard installation

In this guide we are using the path /data/Premium/workspace

Create a docker-compose.yml file with the following content:
```
services:
 wazuh.dashboard:
   image: socacr01.azurecr.io/wazuh-dashboard:4.14.4
   container_name: ${WAZUH_DASHBOARD_CONTAINER_NAME}
   hostname: ${WAZUH_DASHBOARD_HOSTNAME}
   restart: always
   ports:
     - 443:5601
   environment:
     - OPENSEARCH_HOSTS="https://${INDEXER_NODE_1}:9200","https://${INDEXER_NODE_2}:9200","https://${INDEXER_NODE_3}:9200","https://${INDEXER_NODE_4}:9200","https://${INDEXER_NODE_5}:9200"
     - WAZUH_API_URL="https://${WAZUH_MASTER_IP}:55000"
     - API_USERNAME=${API_USERNAME}
     - API_PASSWORD=${API_PASSWORD}
     - DASHBOARD_USERNAME=${DASHBOARD_USERNAME}
     - DASHBOARD_PASSWORD=${DASHBOARD_PASSWORD}
   volumes:
     - ${wazuh_dashboard_pem}:/usr/share/wazuh-dashboard/certs/wazuh-dashboard.pem
     - ${wazuh_dashboard_key_pem}:/usr/share/wazuh-dashboard/certs/wazuh-dashboard-key.pem
     - ${root_ca_pem}:/usr/share/wazuh-dashboard/certs/root-ca.pem
     - ./opensearch_dashboards.yml:/usr/share/wazuh-dashboard/config/opensearch_dashboards.yml
     - ./wazuh.yml:/usr/share/wazuh-dashboard/data/wazuh/config/wazuh.yml
     - ${WAZUH_DASHBOARD_DATA_DIR}/wazuh-dashboard-config:/usr/share/wazuh-dashboard/data/wazuh/config
     - ${WAZUH_DASHBOARD_DATA_DIR}/wazuh-dashboard-custom:/usr/share/wazuh-dashboard/plugins/wazuh/public/assets/custom
```
Create opensearch_dashboards.yml with the following content:
```
server.host: 0.0.0.0
server.port: 5601
opensearch.hosts:
  - https://indexer1_IP_address:9200
  - https://indexer2_IP_address:9200
  - https://indexer3_IP_address:9200
opensearch.ssl.verificationMode: certificate
opensearch.requestHeadersWhitelist: ["securitytenant","Authorization"]
opensearch_security.multitenancy.enabled: true
opensearch_security.readonly_mode.roles: ["kibana_read_only"]
server.ssl.enabled: true
server.ssl.key: "/usr/share/wazuh-dashboard/certs/wazuh-dashboard-key.pem"
server.ssl.certificate: "/usr/share/wazuh-dashboard/certs/wazuh-dashboard.pem"
opensearch.ssl.certificateAuthorities: ["/usr/share/wazuh-dashboard/certs/root-ca.pem"]
uiSettings.overrides.defaultRoute: /app/wz-home
# Session expiration settings
opensearch_security.cookie.ttl: 900000
opensearch_security.session.ttl: 900000
opensearch_security.session.keepalive: true
opensearch_security.multitenancy.tenants.enable_global: true
opensearch_security.multitenancy.tenants.enable_private: true
opensearch_security.multitenancy.tenants.preferred: ["Private", "Global"]
```
Replace the following placeholders in the .env file with your own values:
- replace `indexer1_IP_address`, `indexer2_IP_address` and `indexer3_IP_address`

Create a wazuh.yml file with the following content:
```
hosts:
  - 1513629884013:
      url: "https://10.10.101.36"
      port: 55000
      username: wazuh-wui
      password: "WazuhApi@789!"
      run_as: true
```
Create .env file with the following content:
```
WAZUH_DASHBOARD_CONTAINER_NAME=wazuh.dashboard
WAZUH_DASHBOARD_HOSTNAME=wazuh.dashboard
WAZUH_DASHBOARD_DATA_DIR=/data/Premium/data
INDEXER_NODE_1=indexer1_IP_addredd
INDEXER_NODE_2=indexer2_IP_addredd
INDEXER_NODE_3=indexer3_IP_addredd
INDEXER_NODE_4=indexer4_IP_addredd
INDEXER_NODE_5=indexer5_IP_addredd
WAZUH_MASTER_IP=manager_IP_addredd
API_USERNAME=wazuh-wui
API_PASSWORD=your_api_password
DASHBOARD_USERNAME=kibanaserver
DASHBOARD_PASSWORD=your_kibana_passowrd
root_ca_pem=./certs/root-ca.pem
wazuh_dashboard_pem=./certs/wazuh.dashboard.pem
wazuh_dashboard_key_pem=./certs/wazuh.dashboard-key.pem
```

Replace the following placeholders in the ossec.conf file with your own values:
- replace `indexer1_IP_address`, `indexer2_IP_address`, `indexer3_IP_address`, `indexer4_IP_address` and `indexer5_IP_address` with the IP addresses of your Wazuh indexer nodes.
- replace `manager_IP_address` with the IP address of your Wazuh manager node.
- replace `your_api_password` with the password for the Wazuh API.
- replace `your_kibana_password` with the password for the Wazuh kibana.

Once you have created the docker-compose.yml, ossec.conf, and .env files for Wazuh worker2, and copied the certificates to the certs directory, you can start the Wazuh worker2 container by running the following command in the same directory as the docker-compose.yml file:
```
docker-compose up -d
```
Check the logs to ensure that the Wazuh dashboard container is running correctly:
```
docker-compose logs -f wazuh.dhashboard
```
Wait for 2 minutes and access the dashboard with the help of Dashboard IP address and login with your admin and admin_password