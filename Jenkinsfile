#!/usr/bin/env groovy
pipeline {
    agent { label 'jnk4stl4' }
    options { buildDiscarder(logRotator(numToKeepStr:'5')) }
    parameters {
        string(name: 'ENV', defaultValue: 'mtp_sbx_nonprod')
        string(name: 'Artifact_Source_URL', defaultValue: '{"override_attributes":{"mc_mpass_switchui": {"test":"hello"}}} ')
        string(name: 'ApplicationRole', defaultValue: 'mc_mpass_switchui-sbx')
        string(name: 'numb',defaultValue: '0' )
        string(name: 'Object_Pipeline',defaultValue: 'ChefObjectPipeline/Masterpass/chef-ui-state-chef-objects')
        string(name: 'Cred_ID', defaultValue: 'masterkas-to')
        string(name: 'gitRepo',defaultValue: 'https://globalrepository.mclocal.int/stash/scm/mpass/chef-ui-state.git')
        string(name: 'gitBranch',defaultValue: 'master')
        string(name: 'chef_ID',defaultValue: 'chef_prod_reader',description: 'Credentials ID for chef server')
        }


    stages  {
        stage("Get Env Info"){
            steps{
                deleteDir()
                script {
                scm = [
                  $class: 'GitSCM',
                  userRemoteConfigs: [[url: "${gitRepo}"]],
                  branches: [[name: "${gitBranch}"]],
                  extensions: [],
                  credentialsId: "${Cred_ID}",
                  doGenerateSubmoduleConfigurations: false,
                  submoduleCfg: []
                    ]
                }
                checkout scm
                script {
                    BITBUCKET_URL = sh(returnStdout: true, script: 'git config remote.origin.url').trim().replaceAll("https://",'')
                }
                sh ("echo ${BITBUCKET_URL}")
                sh "ls -lah"
                sh "git checkout ${gitBranch}"
                sh 'git pull'



                sh 'rm chef-deploy.py || echo "chef-deploy not present"'
                python_script()
                stash name: "stage_stash", includes: "*"
           }
        }
        stage ('Ensure in default role A')
        {
            steps{

                unstash "stage_stash"
                configure_knife()
                sh ("knife search node 'chef_environment:${params.ENV} AND role:${params.ApplicationRole}-*' ${env.connectionParams} -i > ./nodes.list")
                sh ("python chef-deploy.py --source '${params.Artifact_Source_URL}' --role '${params.ApplicationRole}' --action 'move_to_A'")

                commit_to_git()
                build job : "${Object_Pipeline}" , wait: true
                //input("Ensure Nodes Converged?")
                verify_if_chef_client()

                stash name: "stage_stash", includes: "*"
            }
        }

        stage ('Update Deployment role') {
            steps{
                unstash "stage_stash"
                sh ("python chef-deploy.py --source '${params.Artifact_Source_URL}' --role '${params.ApplicationRole}' --action 'B-Role'")

                commit_to_git()
                build job : "${Object_Pipeline}" , wait: true
                verify_if_chef_client()

                stash name: "stage_stash", includes: "*"
            }
        }
        stage('Integration Testing Docker - Deactivated')
        {

            steps {
                echo "!!Need Docker Containers support on Host that talks to chef server!!!"
                /*configure_knife()
                sh ("mkdir -p Integration_Testing")
                dir("Integration_Testing") {
                    sh ("knife download --chef-repo-path `pwd` roles/${params.ApplicationRole}-*.json ${env.connectionParams} ")
                    sh ("knife download --chef-repo-path `pwd` environments/${params.ENV}.json ${env.connectionParams} ")
                    sh ("knife download cookbooks/* --chef-repo-path `pwd` ${env.connectionParams} ")
                    echo "Configure Kitchen.yml"
                    kitchen_yml()
                    sh 'ls -lah'
                    sh "kitchen create"
                    sh "kitchen converge"
                    sh "kitchen verify"
                    sh "kitchen destroy"
                }*/
            }
        }
        stage('Move Initial Set of Hosts to Deployment Role')
        {
            steps {

               unstash "stage_stash"
               sh ("python chef-deploy.py --source '${params.Artifact_Source_URL}' --role '${params.ApplicationRole}' --action 'move_to_B' --numb '${params.numb}'")

               commit_to_git()
               build job : "${Object_Pipeline}" , wait: true
               //input("Ensure Nodes Converged?")
               verify_if_chef_client()

               stash name: "stage_stash", includes: "*"
            }
        }
        stage('Move rest of the hosts to Deployment Role')
        {
            steps {
               unstash "stage_stash"
               sh ("python chef-deploy.py --source '${params.Artifact_Source_URL}' --role '${params.ApplicationRole}' --action 'move_to_B'")

               commit_to_git()
               build job : "${Object_Pipeline}" , wait: true
               //input("Ensure Nodes Converged?")
               verify_if_chef_client()

            }
        }
       }
       post {
        success {
            unstash "stage_stash"
            sh ("python chef-deploy.py --source '${params.Artifact_Source_URL}' --role '${params.ApplicationRole}' --action 'A-Role'")
            sh ("python chef-deploy.py --source '${params.Artifact_Source_URL}' --role '${params.ApplicationRole}' --action 'move_to_A'")

            commit_to_git()
            build job : "${Object_Pipeline}" , wait: true
            //input("Ensure Nodes Converged?")
            verify_if_chef_client()

            sh ('echo "Notify Someone of success"')
            /*dir("Integration_Testing") {
                sh "kitchen destroy"
            }*/


        }
        failure {
            unstash "stage_stash"
            sh ('echo "notify someone of failure and preform roll back"')
            sh ("python chef-deploy.py --source '${params.Artifact_Source_URL}' --role '${params.ApplicationRole}' --action 'move_to_A'")

            commit_to_git()
            build job : "${Object_Pipeline}" , wait: true
            //input("Ensure Nodes Converged?")
            verify_if_chef_client()

            sh ('echo "notify of rollback status"')
            /*dir("Integration_Testing") {
                sh "kitchen destroy"
            }*/
        }
        aborted {
            unstash "stage_stash"
            sh ('echo "notify someone of failure and preform roll back"')
            sh ("python chef-deploy.py --source '${params.Artifact_Source_URL}' --role '${params.ApplicationRole}' --action 'move_to_A'")
            commit_to_git()
            build job : "${Object_Pipeline}" , wait: true
            //input("Ensure Nodes Converged?")
            verify_if_chef_client()
            sh ('echo "notify of rollback status"')
            /*dir("Integration_Testing") {
                sh "kitchen destroy"
            }*/
        }
    }
}

void verify_if_chef_client()
{
     configure_knife()

     echo "Remove Old States"
     sh 'rm knife_status || echo "knife_state not present"'
     sh 'rm knife_state_monitor || echo "knife_state_monitor not present"'
     echo "Verification of Chef Client Begins...."

     timeout(time: 45, unit: 'MINUTES') {
           waitUntil {
                sleep 30
                script {
                    broken_nodes_cli = sh(returnStdout: true, script: "knife status 'role:${params.ApplicationRole}-*' --hide-by-mins 58 ${env.connectionParams}  | grep ago | wc -l").trim()
                    echo "Hosts longer then 58 min : ${broken_nodes_cli}"
                    if (broken_nodes_cli != "0" ) {
                        return false
                    }

                    sh "knife status 'role:${params.ApplicationRole}-*' ${env.connectionParams} > knife_status"
                    compare_script()
                    broken_nodes = sh (returnStatus:true , script: "python compare.py" )
                    sh "cat knife_status"
                    echo "Exit Code for compare time : Time flip at 0 , 1 no time flip: \n ${broken_nodes}"
                    if (broken_nodes == 1 ) {
                       return false
                    }
                    return true

                }
           }
     }
}

void configure_knife() {
    withCredentials([ sshUserPrivateKey( credentialsId: "${chef_ID}", usernameVariable: 'CHEF_URL', keyFileVariable: 'KEYPATH')]) {
           echo "knife config..."
           sh "knife ssl fetch ${CHEF_URL}"
           echo "COPY KEY NOW"
           sh "cat ${KEYPATH} > ${env.WORKSPACE}/chef_key"
           env.connectionParams = "--server-url \'${CHEF_URL}\' --user 'reader' --key '${env.WORKSPACE}/chef_key'"
    }
}

void commit_to_git() {

      withCredentials([[$class: 'UsernamePasswordMultiBinding', credentialsId: "${Cred_ID}", usernameVariable: 'GIT_AUTHOR_NAME', passwordVariable: 'GIT_PASSWORD']]) {
                    sh 'echo ${GIT_AUTHOR_NAME} pushing '
                    sh 'git config user.email "user@test.com"'
                    sh 'git config user.name "Jenkins"'
                    sh 'git config push.default simple'
                    sh 'git commit -am "Automated Jenkins Commit" || echo "Nothing To Commit has been detected"'
                    sh "git push https://${GIT_AUTHOR_NAME}:${GIT_PASSWORD}\\@${BITBUCKET_URL}"
      }
}

void kitchen_yml() {
    echo "Save Variable for kitchen file"
                    script {
                    kitchen = """
---
driver:
  name: docker
  use_sudo: false

provisioner:
  name: chef_zero
  roles_path: './roles'
  environments_path: './environments'
  always_update_cookbooks: true
  client_rb:
environment: ENV_PLACEHOLDER

verifier:
  name: inspec

platforms:
  - name: centos-6.9

suites:
    - name: mpass-apache2
      driver: docker
      environment: ENV_PLACEHOLDER
      run_list:
      - role[ROLE_PLACEHOLDER-b]

                    """
                      kitchen = kitchen.replaceAll(/ENV_PLACEHOLDER/,"${params.ENV}")
                      kitchen = kitchen.replaceAll(/ROLE_PLACEHOLDER/,"${params.ApplicationRole}")
                }
  				 echo "${kitchen}"
                 echo "${env.WORKSPACE}"
                 writeFile file: '.kitchen.yml', text: "${kitchen}"
}




void python_script() {
    script {
    PYTHON_SCRIPT = """
from __future__ import print_function
import json
import argparse


def merge(a, b, path=None, update=True):
    if path is None: path = []
    for key in b:
        if key in a:
            if isinstance(a[key], dict) and isinstance(b[key], dict):
                merge(a[key], b[key], path + [str(key)])
            elif a[key] == b[key]:
                pass # same leaf value
            elif isinstance(a[key], list) and isinstance(b[key], list):
                for idx, val in enumerate(b[key]):
                    a[key][idx] = merge(a[key][idx], b[key][idx], path + [str(key), str(idx)], update=update)
            elif update:
                a[key] = b[key]
            else:
                raise Exception('Conflict at %s' % '.'.join(path + [str(key)]))
        else:
            a[key] = b[key]
    return a



def load_node(name):
    return json.load(open("./nodes/"+name+".json"))

def load_env(name):
    return json.load(open("./environments/"+name+".json"))

def load_role(name):
    return json.load(open("./roles/"+name+".json"))

def update_role(name,update):
    role = load_role(name)
    return merge(role,update)

def update_node(name,update):
    node = load_node(name)
    return merge(node,update)

def update_env(name,update):
    env = load_env(name)
    return merge(env,update)

def load_nodes():
    return open("./nodes.list").readlines()


def parse_args():
    parser = argparse.ArgumentParser(description='Process Chef State Change')
    parser.add_argument('--source',help='Provide URL for Artifact you want to deploy.')
    parser.add_argument('--role', help='Provide Role you want to use.')
    parser.add_argument('--action',help='What is it that you are doing ?')
    parser.add_argument('--numb', type=int,help='How many nodes do you want to start with?',default=0)

    args = parser.parse_args()
    return args
def write_json(path,name,data):
    json.dump(data,open(path+name+".json",'w'))

if __name__ == "__main__":
   args =  parse_args()

   if args.action == "B-Role":
        data = json.loads(args.source)
        write_json("./roles/",args.role+"-b",update_role(args.role+"-b",data))

   if args.action == "A-Role":
        data = json.loads(args.source)
        write_json("./roles/", args.role + "-a", update_role(args.role + "-a", data))

   if args.action == "move_to_A":
        nodes_loaded = load_nodes()
        for i in nodes_loaded:
            if args.numb > 0:
                if nodes_loaded.index(i) >= args.numb:
                    break;
            name = i.rstrip()
            data = load_node(name)
            index = None;
            for l in data["run_list"]:
                if args.role in l:
                    index = data["run_list"].index(l)
            data["run_list"][index] = "role[" + args.role + "-a]"
            write_json("./nodes/", name, data)

   if args.action == "move_to_B":
       nodes_loaded = load_nodes()
       for i in nodes_loaded:
           if args.numb > 0:
               if nodes_loaded.index(i) >= args.numb:
                   break;
           name = i.rstrip()
           data = load_node(name)
           index = None;
           for l in data["run_list"]:
               if args.role in l:
                   index = data["run_list"].index(l)
           data["run_list"][index] = "role[" + args.role + "-b]"
           write_json("./nodes/", name, data)
    """
    }
    writeFile file: "chef-deploy.py", text: "${PYTHON_SCRIPT}"
}

void compare_script() {
    script {
        COMPARE = """
import json



def get_output():
    f = open("knife_status")
    data = list()
    for line in f.readlines():
        data.append(line.split(','))
    return data

def get_timestamp(data):
    di = dict()
    for l in data:
        di[l[1]]=dict()
        di[l[1]]["time"]=l[0].split(' ')[0]
    return di


def get_old_status(data):
    try:
        return json.load(open("knife_state_monitor"))
    except:
        json.dump(data,open("knife_state_monitor",'w'))
        return data

def compare():
    data = get_timestamp(get_output())
    old_data = get_old_status(data)

    for i in data:
        if int(data[i]["time"]) < int(old_data[i]["time"]):
            data[i]["done"] = int(1)
        if "done" in old_data[i].keys():
            data[i]["done"] = old_data[i]["done"]


    json.dump(data, open("knife_state_monitor", 'w'))

    for i in data:
        if "done" in data[i].keys():
            if data[i]["done"] != 1:
                exit(1)
        else:
            exit(1)

compare()
"""
        writeFile file: "compare.py", text: "${COMPARE}"
    }
}
