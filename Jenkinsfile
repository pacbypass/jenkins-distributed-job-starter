
pipeline {
    agent {label 'Worker'}
    parameters {
        booleanParam(defaultValue: false, description: 'Enable debug mode', name: 'DEBUG')
        string(name: 'LIMIT', description: 'limit the number if lines read from FILENAME', defaultValue: "40")
        string(name: 'THREADS', description: 'Number of parallel threads to use for starting new jobs, recommended to be $(nproc)', defaultValue: "8")
        string(name: 'CHUNK_SIZE', description: 'amount of lines of FILENAME to be passed to each underlying job', defaultValue: "5")
        string(name: 'FILENAME', description: 'Newline separated file to be ingested and split', defaultValue: "lol.txt")
    }
    options { 
        quietPeriod(0)
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    stages {
        stage("Distributed scanning") {
            steps {
                script {
                    checkout scm

                    // setup variables
                    def threads = params.THREADS.toInteger()
                    def chunk_size = params.CHUNK_SIZE.toInteger()
                                        
                    def targetLists = generateLists(params.FILENAME, params.LIMIT.toInteger())
                    def chunkSize = Math.max(1, (int)(targetLists.size() / threads))
                    def dividedTargetLists = targetLists.collate(chunkSize)
                                        
                    if (params.DEBUG) {
                        println "archiving our lists"
                        writeFile file: 'target_lists.txt', text: targetLists.join('\n')
                        archiveArtifacts artifacts: 'target_lists.txt', fingerprint: true
                    }
                    
                    def parallelScanJobs = [:]

                    // run eet
                    dividedTargetLists.eachWithIndex { targetList, index ->
                        parallelScanJobs["Bucket-${index}"] = {
                            node('specify-your-label-here-for-best-perf') {
                                targetList.collate(chunk_size).each { chunked ->
                                    build job: "run_chunked",
                                        parameters: [
                                            text(name: 'LIST', value: chunked.join('\n')),
                                            booleanParam(name: 'DEBUG', value: params.DEBUG),
                                        ],
                                        propagate: false,
                                        wait: false
                                }
                            }
                        }
                    }
                    
                    parallel parallelScanJobs
                }
            }
        }
    }
}

def generateLists(String filename, limit=null) {
    def list = readFile(filename).readLines().collect { it.trim() }
    if (limit){
        Collections.shuffle(list)
        return list.take(limit)
    }
    return list
}
