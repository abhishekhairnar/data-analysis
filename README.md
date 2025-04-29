{
  "type": "Foreach",
  "foreach": "@variables('FolderQueue')",
  "actions": {
    "List_folder": {
      "type": "OpenApiConnection",
      "inputs": {
        "parameters": {
          "dataset": "https://fiservcorp.sharepoint.com/sites/APACAutomation",
          "id": "@items('Apply_to_each')"
        },
        "host": {
          "apiId": "/providers/Microsoft.PowerApps/apis/shared_sharepointonline",
          "connection": "shared_sharepointonline",
          "operationId": "ListFolder"
        }
      }
    },
    "Apply_to_each_1": {
      "type": "Foreach",
      "foreach": "@body('List_folder')",
      "actions": {
        "Condition": {
          "type": "If",
          "expression": {
            "and": [
              {
                "equals": [
                  "@items('Apply_to_each_1')?['IsFolder']",
                  true
                ]
              }
            ]
          },
          "actions": {
            "Append_to_array_variable": {
              "type": "AppendToArrayVariable",
              "inputs": {
                "name": "FolderQueue",
                "value": "@items('Apply_to_each_1')?['Path']"
              }
            }
          },
          "else": {
            "actions": {
              "Condition_1": {
                "type": "If",
                "expression": {
                  "and": [
                    {
                      "endsWith": [
                        "@items('Apply_to_each_1')?['Name']",
                        ".xls"
                      ]
                    }
                  ]
                },
                "actions": {
                  "Get_file_content": {
                    "type": "OpenApiConnection",
                    "inputs": {
                      "parameters": {
                        "dataset": "https://fiservcorp.sharepoint.com/sites/APACAutomation",
                        "id": "@items('Apply_to_each_1')",
                        "inferContentType": true
                      },
                      "host": {
                        "apiId": "/providers/Microsoft.PowerApps/apis/shared_sharepointonline",
                        "connection": "shared_sharepointonline",
                        "operationId": "GetFileContent"
                      }
                    }
                  },
                  "Create_file": {
                    "type": "OpenApiConnection",
                    "inputs": {
                      "parameters": {
                        "folderPath": "/Abhi",
                        "name": "@items('Apply_to_each_1')?['Name']",
                        "body": "@body('Get_file_content')"
                      },
                      "host": {
                        "apiId": "/providers/Microsoft.PowerApps/apis/shared_onedriveforbusiness",
                        "connection": "shared_onedriveforbusiness",
                        "operationId": "CreateFile"
                      }
                    },
                    "runAfter": {
                      "Get_file_content": [
                        "Succeeded"
                      ]
                    },
                    "runtimeConfiguration": {
                      "contentTransfer": {
                        "transferMode": "Chunked"
                      }
                    }
                  },
                  "Run_script": {
                    "type": "OpenApiConnection",
                    "inputs": {
                      "parameters": {
                        "source": "me",
                        "drive": "b!yKHQhdlNy0OblICENlGTwG7s0rmp5eJMuPEDoPY8yJY2OOzxLzrHQJ-gr8yk5Le_",
                        "file": "@outputs('Create_file')?['body/Path']",
                        "scriptId": "ms-officescript%3A%2F%2Fonedrive_business_itemlink%2F01PO4XEBUTZ3KNVXDQMBDKDZOMJKFD564O"
                      },
                      "host": {
                        "apiId": "/providers/Microsoft.PowerApps/apis/shared_excelonlinebusiness",
                        "connection": "shared_excelonlinebusiness",
                        "operationId": "RunScriptProd"
                      }
                    },
                    "runAfter": {
                      "Create_file": [
                        "Succeeded"
                      ]
                    }
                  },
                  "Get_file_content_1": {
                    "type": "OpenApiConnection",
                    "inputs": {
                      "parameters": {
                        "id": "replace(item()?['Name'],'.xls','.xlsx')",
                        "inferContentType": true
                      },
                      "host": {
                        "apiId": "/providers/Microsoft.PowerApps/apis/shared_onedriveforbusiness",
                        "connection": "shared_onedriveforbusiness",
                        "operationId": "GetFileContent"
                      }
                    },
                    "runAfter": {
                      "Run_script": [
                        "Succeeded"
                      ]
                    }
                  },
                  "Create_file_1": {
                    "type": "OpenApiConnection",
                    "inputs": {
                      "parameters": {
                        "dataset": "https://fiservcorp.sharepoint.com/sites/APACAutomation",
                        "folderPath": "@items('Apply_to_each_1')?['Path']",
                        "name": "replace(item()?['Name'],'.xls','.xlsx')",
                        "body": "@body('Get_file_content_1')"
                      },
                      "host": {
                        "apiId": "/providers/Microsoft.PowerApps/apis/shared_sharepointonline",
                        "connection": "shared_sharepointonline",
                        "operationId": "CreateFile"
                      }
                    },
                    "runAfter": {
                      "Get_file_content_1": [
                        "Succeeded"
                      ]
                    },
                    "runtimeConfiguration": {
                      "contentTransfer": {
                        "transferMode": "Chunked"
                      }
                    }
                  }
                },
                "else": {
                  "actions": {}
                }
              }
            }
          }
        }
      },
      "runAfter": {
        "List_folder": [
          "Succeeded"
        ]
      }
    }
  },
  "runAfter": {
    "Initialize_variable_1": [
      "Succeeded"
    ]
  }
}
